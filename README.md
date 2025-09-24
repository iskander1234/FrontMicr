private static string? ReadClassificationLoose(Dictionary<string, object>? payload, string? storedPayloadJson)
{
    string? fromIncoming = null;
    try
    {
        if (payload is not null)
        {
            // формат: { payload: { processData: { analyst: { classificationCode } } } }
            if (payload.TryGetValue("payload", out var pRaw) && pRaw is not null)
            {
                var p = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pRaw));
                if (p is not null && p.TryGetValue("processData", out var pdRaw) && pdRaw is not null)
                {
                    var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pdRaw));
                    if (pd is not null && pd.TryGetValue("analyst", out var aRaw) && aRaw is not null)
                    {
                        var analyst = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(aRaw));
                        if (analyst != null && analyst.TryGetValue("classificationCode", out var cRaw) && cRaw is not null)
                            fromIncoming = JsonSerializer.Deserialize<string>(JsonSerializer.Serialize(cRaw));
                    }
                }
            }
            // формат: { processData: { analyst: { classificationCode } } }
            if (fromIncoming is null && payload.TryGetValue("processData", out var pd2) && pd2 is not null)
            {
                var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pd2));
                if (pd is not null && pd.TryGetValue("analyst", out var a2) && a2 is not null)
                {
                    var analyst = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(a2));
                    if (analyst != null && analyst.TryGetValue("classificationCode", out var c2) && c2 is not null)
                        fromIncoming = JsonSerializer.Deserialize<string>(JsonSerializer.Serialize(c2));
                }
            }
        }
    }
    catch { /* no-op */ }

    // fallback к сохранённому JSON процесса
    var fromStored = ReadClassificationFromJson(storedPayloadJson);

    static string? Canon(string? s)
    {
        if (string.IsNullOrWhiteSpace(s)) return null;
        var n = s.Trim().ToLowerInvariant();
        return n switch
        {
            "emergency" => "Emergency",
            "standard"  => "Standard",
            "normal"    => "Normal",
            _           => null
        };
    }

    return Canon(fromIncoming) ?? Canon(fromStored);
}











if (string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
{
    var claimed = await camundaService.CamundaClaimedTasks(...);

    var variables = new Dictionary<string, object>();



    // Гарантия для первой схемы: если нашли — всегда отправляем
    var classification = ReadClassificationLoose(command.PayloadJson, processData.PayloadJson);
    if (!string.IsNullOrWhiteSpace(classification))
        variables["classification"] = classification;
