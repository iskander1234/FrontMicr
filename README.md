var recipients = ExtractFromPayloadFlexible<UserDataDto>(payloadDict, key) ?? new();
bool isSequential = ReadIsSequentialFromProcessData(payloadDict)


// ---- VALIDATION: только для последовательного согласования ----
if (request.Stage == ProcessStage.Approval && isSequential)
{
    // 1) Проверяем, что у всех есть корректный Order (>0)
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

    // 2) Проверяем уникальность значений Order
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
}



// ----- VALIDATION: только для последовательного согласования -----
if (request.Stage == ProcessStage.Approval && isSequential)
{
    // 1) У каждого участника должен быть Order > 0
    var invalid = recipients.Where(r => !r.Order.HasValue || r.Order.Value <= 0).ToList();
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

    // 2) Значения Order должны быть уникальны
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

    // (опционально) 3) Если хочешь требовать непрерывность 1..N, раскомментируй:
    // var orders = recipients.Select(r => r.Order!.Value).OrderBy(x => x).ToArray();
    // for (int i = 0; i < orders.Length; i++)
    //     if (orders[i] != i + 1)
    //         throw new HandlerException(
    //            $"В последовательном режиме порядок должен быть непрерывным: 1..{orders.Length}. Найдено: {string.Join(", ", orders)}.",
    //            ErrorCodesEnum.Business);
}







///if (request.Stage == ProcessStage.Approval && isSequential)



// Последовательный режим: первый — Pending, остальные — Waiting, порядок строго по Order
var ordered = recipients
    .OrderBy(r => r.Order!.Value)  // важно: уже провалидировали, что Order есть
    .ToList();

bool isFirst = true;
foreach (var r in ordered)
{
    var assignee = r.UserCode; // loginAD -> UserCode (как у тебя было)
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
        Order          = r.Order!.Value // используем исходный Order (не перенумеровываем)
    }, cancellationToken);

    isFirst = false;
}
