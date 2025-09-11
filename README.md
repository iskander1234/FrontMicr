// ★ Rework: если отправляем дальше (Submit) и в JSON есть regData.regnum — просто делаем update всего ProcessData
if (Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var currentStage)
    && currentStage == ProcessStage.Rework
    && command.Action == ProcessAction.Submit)
{
    // Берём JSON из команды, если прислали новый; иначе — текущий из ProcessData
    var srcJson = !string.IsNullOrWhiteSpace(command.PayloadJson)
        ? command.PayloadJson
        : processData.PayloadJson;

    if (!string.IsNullOrWhiteSpace(srcJson) && JsonHasRegnum(srcJson))
    {
        // Фиксируем все изменения целиком — перезаписываем PayloadJson только если пришёл новый
        if (!string.IsNullOrWhiteSpace(command.PayloadJson))
            processData.PayloadJson = command.PayloadJson;

        await unitOfWork.ProcessDataRepository.UpdateAsync(cancellationToken, processData);
        // Ничего больше не меняем; ниже идёт основная логика как была
    }
}



private static bool JsonHasRegnum(string json)
{
    try
    {
        using var doc = JsonDocument.Parse(json);
        var root = doc.RootElement;

        if (root.ValueKind != JsonValueKind.Object) return false;

        if (TryGetPropertyCaseInsensitive(root, "regData", out var regDataEl) &&
            TryGetPropertyCaseInsensitive(regDataEl, "regnum", out var regnumEl) &&
            regnumEl.ValueKind == JsonValueKind.String &&
            !string.IsNullOrWhiteSpace(regnumEl.GetString()))
        {
            return true;
        }
    }
    catch
    {
        // некорректный JSON — считаем, что regnum нет
    }
    return false;
}

private static bool TryGetPropertyCaseInsensitive(JsonElement element, string name, out JsonElement value)
{
    if (element.ValueKind == JsonValueKind.Object)
    {
        foreach (var p in element.EnumerateObject())
        {
            if (string.Equals(p.Name, name, StringComparison.OrdinalIgnoreCase))
            {
                value = p.Value;
                return true;
            }
        }
    }
    value = default;
    return false;
}
