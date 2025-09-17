public class SendAcquaintanceCommandHandler(
    IUnitOfWork unitOfWork,
    IProcessTaskService processTaskService
) : IRequestHandler<SendAcquaintanceCommand, BaseResponseDto<SendProcessResponse>>
{
    private static readonly string BlockCode = ProcessStage.Acquaintance.ToString();

    public async Task<BaseResponseDto<SendProcessResponse>> Handle(SendAcquaintanceCommand cmd, CancellationToken ct)
    {
        var pd = await unitOfWork.ProcessDataRepository.GetByIdAsync(ct, cmd.ProcessDataId)
                 ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

        var proc = await unitOfWork.ProcessRepository.GetByFilterAsync(ct, p => p.ProcessCode == pd.ProcessCode);

        // НЕ ДОЛЖНО БЫТЬ NULL — готовим инициатора с фолбэками
        var initCode = string.IsNullOrWhiteSpace(pd.InitiatorCode) ? "system" : pd.InitiatorCode!;
        var initName = string.IsNullOrWhiteSpace(pd.InitiatorName) ? initCode : pd.InitiatorName!;

        var (recipients, comment) = TryReadFromPayload(cmd.PayloadJson);
        if (recipients.Count == 0)
            throw new HandlerException("В payload.acquaintance.recipients не найден ни один получатель",
                ErrorCodesEnum.Business);

        var existingPending = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
            ct,
            t => t.ProcessDataId == pd.Id
                 && t.BlockCode == BlockCode
                 && t.Status == "Pending"
                 && !string.IsNullOrEmpty(t.AssigneeCode));

        var already = existingPending.Select(t => t.AssigneeCode!).ToHashSet(StringComparer.OrdinalIgnoreCase);
        var toCreate = recipients.Where(r => !already.Contains(r.UserCode)).ToList();
        if (toCreate.Count == 0)
            return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };

        // родитель (скрытый) — ВАЖНО: инициатор должен быть заполнен
        var parentId = Guid.NewGuid();
        await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
        {
            EntityId      = parentId,
            ProcessDataId = pd.Id,
            ProcessCode   = pd.ProcessCode,
            ProcessName   = proc?.ProcessName,
            RegNumber     = pd.RegNumber,
            Title         = pd.Title,
            BlockCode     = BlockCode,
            BlockName     = "Ознакомление",
            AssigneeCode  = "system",
            AssigneeName  = "system",
            Status        = "Hidden",
            Comment       = comment,

            // <<< ДОБАВЛЕНО
            InitiatorCode = initCode,
            InitiatorName = initName
        }, ct);

        // дочерние Pending-задачи — тоже обязательно с инициатором
        foreach (var r in toCreate)
        {
            var code = r.UserCode?.Trim();
            if (string.IsNullOrWhiteSpace(code)) continue;
            var name = string.IsNullOrWhiteSpace(r.UserName) ? code : r.UserName!.Trim();

            await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
            {
                EntityId      = Guid.NewGuid(),
                ParentTaskId  = parentId,
                ProcessDataId = pd.Id,
                ProcessCode   = pd.ProcessCode,
                ProcessName   = proc?.ProcessName,
                RegNumber     = pd.RegNumber,
                Title         = pd.Title,
                BlockCode     = BlockCode,
                BlockName     = "Ознакомление",
                AssigneeCode  = code,
                AssigneeName  = name,
                Status        = "Pending",
                Comment       = comment,

                // <<< ДОБАВЛЕНО
                InitiatorCode = initCode,
                InitiatorName = initName
            }, ct);

            await processTaskService.RefreshUserTaskCacheAsync(code, ct);
        }

        await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(new ProcessTaskHistoryCreatedEvent
        {
            ProcessDataId = pd.Id,
            TaskId        = parentId,
            BlockCode     = BlockCode,
            BlockName     = "Ознакомление",
            Action        = "SendForAcquaintance",
            ActionName    = "Отправлено на ознакомление",
            Timestamp     = DateTime.Now,
            Description   = string.Join(", ", toCreate.Select(x => $"{x.UserName ?? x.UserCode}({x.UserCode})")),

            // история обычно тоже хранит инициатора — не помешает
            InitiatorCode = initCode,
            InitiatorName = initName
        }, ct);

        return new BaseResponseDto<SendProcessResponse>
        {
            Data = new SendProcessResponse { Success = true }
        };
    }

    private static (List<UserInfo> Recipients, string? Comment) TryReadFromPayload(Dictionary<string, object>? payload)
    {
        var list = new List<UserInfo>();
        string? comment = null;
        if (payload is null || payload.Count == 0) return (list, null);

        try
        {
            var json = JsonSerializer.Serialize(payload);
            var root = JsonNode.Parse(json)!.AsObject();
            var acq  = root["acquaintance"] as JsonObject;
            if (acq is null) return (list, null);

            comment = acq["comment"]?.GetValue<string>() ?? acq["message"]?.GetValue<string>();
            if (acq["recipients"] is JsonArray arr)
            {
                foreach (var n in arr.OfType<JsonObject>())
                {
                    var code = n["userCode"]?.GetValue<string>()?.Trim();
                    if (string.IsNullOrWhiteSpace(code)) continue;
                    var name = n["userName"]?.GetValue<string>();
                    list.Add(new UserInfo { UserCode = code!, UserName = name });
                }
            }
        }
        catch { /* ignore */ }

        list = list.GroupBy(x => x.UserCode, StringComparer.OrdinalIgnoreCase).Select(g => g.First()).ToList();
        return (list, comment);
    }
