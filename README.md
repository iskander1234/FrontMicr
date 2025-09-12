if (!Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var stageForCamunda))
    throw new InvalidOperationException($"Unknown process stage: {currentTask.BlockCode}");


// Сабмитим parentTask => переменную считаем по parentTask.BlockCode
if (!Enum.TryParse<ProcessStage>(parentTask.BlockCode, out var stageForCamunda))
    throw new InvalidOperationException($"Unknown process stage: {parentTask.BlockCode}");



if (!string.Equals(mergedJson, processData.PayloadJson, StringComparison.Ordinal))
{
    await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataEditedEvent { ... }, cancellationToken);
    processData.PayloadJson = mergedJson;
}


if (!JsonSemanticallyEqual(mergedJson, processData.PayloadJson))
{
    await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataEditedEvent
    {
        EntityId      = processData.Id,
        StatusCode    = processData.StatusCode,
        StatusName    = processData.StatusName,
        PayloadJson   = mergedJson,
        InitiatorCode = processData.InitiatorCode,
        InitiatorName = processData.InitiatorName,
        Title         = processData.Title
        // StartedAt не трогаем
    }, cancellationToken);

    processData.PayloadJson = mergedJson; // дальше читаем уже актуальный JSON
}


private static bool JsonSemanticallyEqual(string? a, string? b)
{
    if (string.IsNullOrWhiteSpace(a) && string.IsNullOrWhiteSpace(b)) return true;
    if (string.IsNullOrWhiteSpace(a) || string.IsNullOrWhiteSpace(b)) return false;
    try
    {
        using var ja = JsonDocument.Parse(a);
        using var jb = JsonDocument.Parse(b);
        return ja.RootElement.ValueEquals(jb.RootElement);
    }
    catch
    {
        return false; // если не парсится — считаем разными
    }
}
