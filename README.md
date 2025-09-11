
// Rework: если отправляем дальше (Submit) и в payload уже есть regData.regnum — обновим RegNumber
if (Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var currentStage)
    && currentStage == ProcessStage.Rework
    && command.Action == ProcessAction.Submit)
{
    var regnum = TryReadRegnumFromPayloadJson(processData.PayloadJson);
    if (!string.IsNullOrWhiteSpace(regnum)
        && !string.Equals(processData.RegNumber, regnum, StringComparison.Ordinal))
    {
        processData.RegNumber = regnum;
        await unitOfWork.ProcessDataRepository.UpdateAsync(cancellationToken, processData);
        // ничего больше не меняем, логика ниже идёт как была
    }
}



private static string? TryReadRegnumFromPayloadJson(string? payloadJson)
{
    if (string.IsNullOrWhiteSpace(payloadJson)) return null;

    try
    {
        using var doc = JsonDocument.Parse(payloadJson);
        var root = doc.RootElement;

        if (TryGetPropertyCaseInsensitive(root, "regData", out var regDataEl) &&
            TryGetPropertyCaseInsensitive(regDataEl, "regnum", out var regnumEl) &&
            regnumEl.ValueKind == JsonValueKind.String)
        {
            return regnumEl.GetString();
        }
    }
    catch
    {
        // некорректный JSON — игнорируем, вернём null
    }

    return null;
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
