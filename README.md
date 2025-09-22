private static string BuildTitle(string oldTitle, string mergedJson)
{
    try
    {
        var root = JsonNode.Parse(mergedJson)?.AsObject();
        var explicitTitle = root?["title"]?.GetValue<string>();
        if (!string.IsNullOrWhiteSpace(explicitTitle))
            return explicitTitle!;

        var regnum = root?["regData"]?["regnum"]?.GetValue<string>();
        if (!string.IsNullOrWhiteSpace(regnum))
        {
            // если regnum уже есть в старом заголовке — не дублируем
            if (!string.IsNullOrWhiteSpace(oldTitle) &&
                oldTitle.Contains(regnum, StringComparison.OrdinalIgnoreCase))
                return oldTitle;

            return string.IsNullOrWhiteSpace(oldTitle)
                ? $"Заявка № {regnum}"
                : $"{oldTitle} (№ {regnum})";
        }
    }
    catch { /* no-op */ }

    return oldTitle;
}


if (!string.Equals(mergedJson, processData.PayloadJson, StringComparison.Ordinal))
{
    var newTitle = BuildTitle(processData.Title, mergedJson);

    await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataEditedEvent
    {
        EntityId = processData.Id,
        StatusCode = processData.StatusCode,
        StatusName = processData.StatusName,
        PayloadJson = mergedJson,
        InitiatorCode = processData.InitiatorCode,
        InitiatorName = processData.InitiatorName,
        Title = newTitle
        // StartedAt не трогаем
    }, cancellationToken);

    // файлы — по новому payload
    await ReconcileFilesAsync(processData.Id, mergedJson, unitOfWork, cancellationToken);

    // держим актуальную версию локально — ВКЛЮЧАЯ Title
    processData.PayloadJson = mergedJson;
    processData.Title = newTitle;
}
