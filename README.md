public class SendAcquaintanceCommandHandler(
    IUnitOfWork unitOfWork,
    IProcessTaskService processTaskService
) : IRequestHandler<SendAcquaintanceCommand, BaseResponseDto<SendProcessResponse>>
{
    private const string AcquaintanceCode = "Acquaintance";

    public async Task<BaseResponseDto<SendProcessResponse>> Handle(SendAcquaintanceCommand cmd, CancellationToken ct)
    {
        var pd = await unitOfWork.ProcessDataRepository.GetByIdAsync(ct, cmd.ProcessDataId)
                 ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

        var proc = await unitOfWork.ProcessRepository.GetByFilterAsync(ct, p => p.ProcessCode == pd.ProcessCode);

        // 1) читаем получателей из payloadJson
        var (recipients, comment) = TryReadFromPayload(cmd.PayloadJson);
        if (recipients.Count == 0)
            throw new HandlerException("В payload.acquaintance.recipients нет получателей", ErrorCodesEnum.Business);

        // идемпотентность: не дублируем уже висящие Pending
        var existingPending = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
            ct, t => t.ProcessDataId == pd.Id && t.BlockCode == AcquaintanceCode &&
                     t.Status == "Pending" && !string.IsNullOrEmpty(t.AssigneeCode));
        var already = existingPending.Select(x => x.AssigneeCode!).ToHashSet(StringComparer.OrdinalIgnoreCase);
        var toCreate = recipients.Where(r => !already.Contains(r.UserCode)).ToList();
        if (toCreate.Count == 0)
            return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };

        // 2) создаём "родителя" группировки (НЕ system, а инициатор)
        var parentId = Guid.NewGuid();
        await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
        {
            EntityId      = parentId,
            ProcessDataId = pd.Id,
            ProcessCode   = pd.ProcessCode,
            ProcessName   = proc?.ProcessName,
            RegNumber     = pd.RegNumber,
            Title         = pd.Title,
            BlockCode     = AcquaintanceCode,
            BlockName     = "Ознакомление",
            Stage         = ProcessStage.Acquaintance.ToString(),
            Status        = "Hidden",
            Comment       = comment,

            // чтобы не было "system"
            AssigneeCode  = pd.InitiatorCode,
            AssigneeName  = pd.InitiatorName,

            // если у тебя в БД InitiatorCode/Name not-null — тоже проставь
            InitiatorCode = pd.InitiatorCode,
            InitiatorName = pd.InitiatorName
        }, ct, autoCommit: true); // ВАЖНО: зафиксировать родителя раньше детей

        // (подстраховка: читаем фактического родителя)
        var parent = await unitOfWork.ProcessTaskRepository.GetByIdAsync(ct, parentId)
                    ?? throw new HandlerException("Не удалось создать родителя ознакомления", ErrorCodesEnum.Business);

        // 3) создаём дочерние Pending для каждого получателя
        foreach (var r in toCreate)
        {
            var assigneeName = string.IsNullOrWhiteSpace(r.UserName) ? r.UserCode : r.UserName;

            await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
            {
                EntityId      = Guid.NewGuid(),
                ParentTaskId  = parent.Id,          // уже существует в БД
                ProcessDataId = pd.Id,
                ProcessCode   = pd.ProcessCode,
                ProcessName   = proc?.ProcessName,
                RegNumber     = pd.RegNumber,
                Title         = pd.Title,
                BlockCode     = AcquaintanceCode,
                BlockName     = "Ознакомление",
                Stage         = ProcessStage.Acquaintance.ToString(),
                Status        = "Pending",
                Comment       = comment,

                // вот тут — строго из payload
                AssigneeCode  = r.UserCode,         // "m.ilespayev"
                AssigneeName  = assigneeName,       // "Илеспаев Меииржан Анварович"

                // not-null поля
                InitiatorCode = pd.InitiatorCode,
                InitiatorName = pd.InitiatorName
            }, ct);

            await processTaskService.RefreshUserTaskCacheAsync(r.UserCode, ct);
        }

        // 4) история
        await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(new ProcessTaskHistoryCreatedEvent
        {
            ProcessDataId = pd.Id,
            TaskId        = parent.Id,
            BlockCode     = AcquaintanceCode,
            BlockName     = "Ознакомление",
            Action        = "SendForAcquaintance",
            ActionName    = "Отправлено на ознакомление",
            Timestamp     = DateTime.Now,
            Description   = string.Join(", ", toCreate.Select(x => $"{x.UserName}({x.UserCode})")),
            InitiatorCode = pd.InitiatorCode,
            InitiatorName = pd.InitiatorName
        }, ct);

        return new BaseResponseDto<SendProcessResponse>
        {
            Data = new SendProcessResponse { Success = true }
        };
    }

    private static (List<UserInfo> Recipients, string? Comment) TryReadFromPayload(Dictionary<string, object>? payload)
    {
        var recipients = new List<UserInfo>();
        string? comment = null;
        if (payload == null || payload.Count == 0) return (recipients, null);

        try
        {
            var json = JsonSerializer.Serialize(payload);
            var root = JsonNode.Parse(json)!.AsObject();
            var acq  = root["acquaintance"] as JsonObject;
            if (acq is null) return (recipients, null);

            comment = acq["comment"]?.GetValue<string>() ?? acq["message"]?.GetValue<string>();

            if (acq["recipients"] is JsonArray arr)
            {
                foreach (var n in arr.OfType<JsonObject>())
                {
                    var code = n["userCode"]?.GetValue<string>()?.Trim();
                    if (string.IsNullOrWhiteSpace(code)) continue;
                    var name = n["userName"]?.GetValue<string>();
                    recipients.Add(new UserInfo { UserCode = code!, UserName = name });
                }
            }
        }
        catch { /* ignore */ }

        // dedupe
        recipients = recipients
            .GroupBy(x => x.UserCode, StringComparer.OrdinalIgnoreCase)
            .Select(g => g.First())
            .ToList();

        return (recipients, comment);
    }
}
