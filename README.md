private static string? ReadClassificationFromJson(string? json)
{
    if (string.IsNullOrWhiteSpace(json)) return null;

    try
    {
        using var doc = JsonDocument.Parse(json);
        var root = doc.RootElement;

        static bool TryGetPropertyCaseInsensitive(JsonElement obj, string name, out JsonElement value)
        {
            if (obj.ValueKind == JsonValueKind.Object)
            {
                foreach (var p in obj.EnumerateObject())
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

        // processData.analyst.classificationCode
        if (TryGetPropertyCaseInsensitive(root, "processData", out var pd) &&
            TryGetPropertyCaseInsensitive(pd, "analyst", out var analyst) &&
            TryGetPropertyCaseInsensitive(analyst, "classificationCode", out var cc) &&
            cc.ValueKind == JsonValueKind.String)
        {
            return Canon(cc.GetString());
        }

        // плоские варианты в корне
        string? flat =
            (TryGetPropertyCaseInsensitive(root, "classification", out var c) && c.ValueKind == JsonValueKind.String ? c.GetString() : null)
            ?? (TryGetPropertyCaseInsensitive(root, "itsmLevel", out var il) && il.ValueKind == JsonValueKind.String ? il.GetString() : null)
            ?? (TryGetPropertyCaseInsensitive(root, "level", out var lvl) && lvl.ValueKind == JsonValueKind.String ? lvl.GetString() : null)
            ?? (TryGetPropertyCaseInsensitive(root, "priority", out var pr) && pr.ValueKind == JsonValueKind.String ? pr.GetString() : null);

        if (flat is not null) return Canon(flat);

        // плоские варианты внутри processData (на всякий случай)
        if (pd.ValueKind == JsonValueKind.Object)
        {
            flat =
                (TryGetPropertyCaseInsensitive(pd, "classification", out var c2) && c2.ValueKind == JsonValueKind.String ? c2.GetString() : null)
                ?? (TryGetPropertyCaseInsensitive(pd, "itsmLevel", out var il2) && il2.ValueKind == JsonValueKind.String ? il2.GetString() : null)
                ?? (TryGetPropertyCaseInsensitive(pd, "level", out var lvl2) && lvl2.ValueKind == JsonValueKind.String ? lvl2.GetString() : null)
                ?? (TryGetPropertyCaseInsensitive(pd, "priority", out var pr2) && pr2.ValueKind == JsonValueKind.String ? pr2.GetString() : null);

            if (flat is not null) return Canon(flat);
        }
    }
    catch { /* ignore */ }

    return null;
}



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
    catch { /* ignore */ }

    // fallback: берём из сохранённого payload процесса
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



var variables = new Dictionary<string, object>();

// гарантируем переменную classification для первой схемы
var classification = ReadClassificationLoose(command.PayloadJson, processData.PayloadJson);
if (!string.IsNullOrWhiteSpace(classification))
    variables["classification"] = classification;

// ... далее твои Line1/Line2 переменные и submit
