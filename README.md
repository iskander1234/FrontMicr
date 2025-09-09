// ... твой код выше без изменений ...

var recipients = ExtractFromPayloadFlexible<UserDataDto>(payloadDict, key) ?? new();

// NEW: пустой список — сразу бизнес-ошибка, чтобы не создавать «пустую» стадию
if (recipients.Count == 0)
{
    throw new HandlerException(
        $"Для этапа {request.Stage} не переданы исполнители в поле '{key}'.",
        ErrorCodesEnum.Business);
}

// 1) читаем тип согласования из processData.approvalTypeCode
bool isSequential = ReadIsSequentialFromProcessData(payloadDict);

// ---- VALIDATION: только для последовательного согласования ----
if (request.Stage == ProcessStage.Approval && isSequential)
{
    // уже было: Order > 0
    var invalid = recipients
        .Where(r => !r.Order.HasValue || r.Order!.Value <= 0)
        .ToList();

    if (invalid.Any())
    {
        var who = string.Join(", ",
            invalid.Select(r => string.IsNullOrWhiteSpace(r.UserName) ? r.UserCode : r.UserName)
                   .Where(s => !string.IsNullOrWhiteSpace(s))
                   .Distinct()
                   .Take(5));
        var tail = invalid.Count > 5 ? $" и ещё {invalid.Count - 5}" : "";
        throw new HandlerException(
            $"В последовательном режиме требуется указать порядок (Order > 0) для каждого участника. Не указан у: {who}{tail}.",
            ErrorCodesEnum.Business);
    }

    // уже было: уникальность Order
    var duplicates = recipients
        .GroupBy(r => r.Order!.Value)
        .Where(g => g.Count() > 1)
        .Select(g => g.Key)
        .OrderBy(x => x)
        .ToList();

    if (duplicates.Any())
    {
        throw new HandlerException(
            $"В последовательном режиме значения Order должны быть уникальны. Дублируются: {string.Join(", ", duplicates)}.",
            ErrorCodesEnum.Business);
    }

    // NEW (опционально, но очень полезно): требуем непрерывность 1..N
    var orders = recipients.Select(r => r.Order!.Value).OrderBy(x => x).ToArray();
    for (int i = 0; i < orders.Length; i++)
    {
        if (orders[i] != i + 1)
            throw new HandlerException(
                $"В последовательном режиме порядок должен быть непрерывным: 1..{orders.Length}. Найдено: {string.Join(", ", orders)}.",
                ErrorCodesEnum.Business);
    }
}

// NEW: идемпотентность. Если по этой заявке и стадии уже есть задачи — не создаём повторно
var stageHasTasks = await unitOfWork.ProcessTaskRepository.CountAsync(
    cancellationToken,
    t => t.ProcessDataId == processData.Id && t.BlockCode == processStageInfo.Code
) > 0;

if (stageHasTasks)
{
    // Мы уже сделали ProcessDataBlockChangedEvent выше — остаётся просто выйти без дублей.
    return new BaseResponseDto<SetProcessStageResponse>
    {
        Data = new SetProcessStageResponse { Status = "Ok" }
    };
}

// 2) родительская system-задача — как раньше
var systemTask = await processTaskService.CreateSystemTaskAsync(
    processStageInfo.Code, processStageInfo.Name, processData, cancellationToken);

// 3) Если Approval и режим Sequentially — создаём детей Pending/Waiting и пишем Order
if (request.Stage == ProcessStage.Approval && isSequential)
{
    var ordered = recipients
        .OrderBy(r => r.Order!.Value)  // уже провалидировали наличие Order
        .ToList();

    bool isFirst = true;
    foreach (var r in ordered)
    {
        var assignee = r.UserCode;
        if (string.IsNullOrWhiteSpace(assignee)) continue;

        await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
        {
            EntityId       = Guid.NewGuid(),
            ProcessDataId  = processData.Id,
            ParentTaskId   = systemTask.EntityId,

            ProcessCode    = processData.ProcessCode,
            ProcessName    = processData.ProcessName ?? "",
            RegNumber      = processData.RegNumber ?? "",
            InitiatorCode  = processData.InitiatorCode ?? "",
            InitiatorName  = processData.InitiatorName ?? "",
            Title          = processData.Title ?? "",

            AssigneeCode   = assignee.Trim().ToLowerInvariant(),
            AssigneeName   = r.UserName ?? "",
            BlockCode      = processStageInfo.Code,
            BlockName      = processStageInfo.Name,
            Status         = isFirst ? "Pending" : "Waiting",
            Order          = r.Order!.Value
        }, cancellationToken);

        isFirst = false;
    }
}
else
{
    // параллельно — как было
    await processTaskService.CreateTasksNewAsync(
        processStageInfo.Code, processStageInfo.Name,
        recipients, processData, systemTask.EntityId, cancellationToken);
}

// ... твой код ниже без изменений ...
