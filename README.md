// Rework → Submit: апдейтим ProcessData ТОЛЬКО если в payload есть regData.regnum
if (Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var currentStage)
    && currentStage == ProcessStage.Rework
    && command.Action == ProcessAction.Submit)
{
    // payload может не прийти — тогда апдейт просто пропускаем
    if (command.PayloadJson is not null)
    {
        var incomingJson = JsonSerializer.Serialize(command.PayloadJson);

        // ← ключевое условие: обновляем только если есть regData.regnum
        if (JsonHasRegnum(incomingJson))
        {
            var mergedJson = MergePayloadPreserve(processData.PayloadJson, incomingJson);

            // апдейтим только при реальных изменениях
            if (!string.Equals(mergedJson, processData.PayloadJson, StringComparison.Ordinal))
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

                // держим актуальную версию локально, чтобы ниже читалась новая
                processData.PayloadJson = mergedJson;
            }
        }
        // если regnum нет — апдейт пропускаем, идём дальше по вашей логике
    }
}



private static bool JsonHasRegnum(string json)
{
    try
    {
        using var doc = JsonDocument.Parse(json);
        var root = doc.RootElement;

        if (root.ValueKind != JsonValueKind.Object) return false;

        if (root.TryGetProperty("regData", out var rd) || 
            TryGetPropertyCaseInsensitive(root, "regData", out rd))
        {
            if (rd.ValueKind == JsonValueKind.Object &&
                (rd.TryGetProperty("regnum", out var rn) || TryGetPropertyCaseInsensitive(rd, "regnum", out rn)))
            {
                return rn.ValueKind == JsonValueKind.String &&
                       !string.IsNullOrWhiteSpace(rn.GetString());
            }
        }
    }
    catch { /* некорректный JSON — считаем, что regnum нет */ }

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




