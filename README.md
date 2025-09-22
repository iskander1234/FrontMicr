// вместо ReadItsmLevel(...)
static string? ReadClassification(string json)
{
    // приоритет: processData.analyst.classificationCode
    var lvl = TryRead<string>(json, "processData", "analyst", "classificationCode")
              ?? TryRead<string>(json, "processData", "classification")
              ?? TryRead<string>(json, "processData", "level")
              ?? TryRead<string>(json, "processData", "priority");

    if (string.IsNullOrWhiteSpace(lvl)) return null;

    var v = lvl.Trim();
    return v.Equals("Emergency", StringComparison.OrdinalIgnoreCase) ? "Emergency" :
           v.Equals("Standard",  StringComparison.OrdinalIgnoreCase) ? "Standard"  :
           v.Equals("Normal",    StringComparison.OrdinalIgnoreCase) ? "Normal"    :
           null;
}



өөө

var vars = new Dictionary<string, object?>(StringComparer.Ordinal)
{
    ["processGuid"]   = created.EntityId.ToString(),
    ["initiatorCode"] = string.IsNullOrWhiteSpace(command.InitiatorCode) ? null : command.InitiatorCode,
    ["initiatorName"] = string.IsNullOrWhiteSpace(command.InitiatorName) ? null : command.InitiatorName
};

var classification = ReadClassification(payloadJson);
if (!string.IsNullOrWhiteSpace(classification))
    vars["classification"] = classification;

var cleaned = vars.Where(kv => kv.Value is not null)
                  .ToDictionary(kv => kv.Key, kv => kv.Value!);

if (cleaned.Values.Any(v => v is IDictionary<string, object>))
    throw new HandlerException("Camunda variables must be plain key/value.", ErrorCodesEnum.Business);

var pi = await camundaService.CamundaStartProcess(new CamundaStartProcessRequest
{
    processCode = command.ProcessCode,
    variables   = cleaned
});





private static Dictionary<string, object> BuildItsmLevelVariables(Dictionary<string, object>? payload)
{
    if (payload is null) return new();

    static string? ReadStr(Dictionary<string, object> p, string key)
    {
        if (!p.TryGetValue(key, out var v) || v is null) return null;
        var json = JsonSerializer.Serialize(v);
        return JsonSerializer.Deserialize<string>(json, new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
    }

    // поддержка вложенности: processData -> analyst -> classificationCode
    string? fromAnalyst = null;
    if (payload.TryGetValue("processData", out var pdRaw) && pdRaw is not null)
    {
        var pdJson = JsonSerializer.Serialize(pdRaw);
        var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(pdJson,
            new JsonSerializerOptions { PropertyNameCaseInsensitive = true });

        if (pd != null && pd.TryGetValue("analyst", out var aRaw) && aRaw is not null)
        {
            var aJson = JsonSerializer.Serialize(aRaw);
            var analyst = JsonSerializer.Deserialize<Dictionary<string, object>>(aJson,
                new JsonSerializerOptions { PropertyNameCaseInsensitive = true });

            if (analyst != null)
                fromAnalyst = ReadStr(analyst, "classificationCode");
        }
    }

    var level = fromAnalyst
                ?? ReadStr(payload, "classification")    // на всякий случай плоский
                ?? ReadStr(payload, "classifacation")    // опечатка
                ?? ReadStr(payload, "itsmLevel")
                ?? ReadStr(payload, "level")
                ?? ReadStr(payload, "priority");

    if (string.IsNullOrWhiteSpace(level)) return new();

    var canonical = level.Trim().ToLowerInvariant() switch
    {
        "emergency" => "Emergency",
        "standard"  => "Standard",
        "normal"    => "Normal",
        _ => null
    };

    return canonical is null ? new() : new() { ["classification"] = canonical };
}

