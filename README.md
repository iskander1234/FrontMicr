// было
var pi = await camundaService.CamundaStartProcess(new CamundaStartProcessRequest
{
    processCode = command.ProcessCode,
    variables   = new Dictionary<string, object> { { "processGuid", created.EntityId.ToString() } }
});


var vars = CamundaVars(
    ("processGuid",   created.EntityId.ToString(), "String"),
    ("initiatorCode", command.InitiatorCode,       "String"),
    ("initiatorName", command.InitiatorName,       "String"),
    // если есть приоритет/уровень — отдаём (англ. вариант, как договаривались)
    ("itsmLevel",     ReadItsmLevel(payloadJson),  "String") // null будет проигнорирован
);

var pi = await camundaService.CamundaStartProcess(new CamundaStartProcessRequest
{
    processCode = command.ProcessCode,
    businessKey = created.EntityId.ToString(),   // <— очень желательно
    variables   = vars
});

// helperы рядом с хендлером
static Dictionary<string, object> CamundaVars(params (string name, object? value, string type)[] items)
{
    var d = new Dictionary<string, object>(StringComparer.Ordinal);
    foreach (var it in items)
    {
        if (it.value is null) continue; // пропускаем пустые
        d[it.name] = new Dictionary<string, object>
        {
            ["value"] = it.value,
            ["type"]  = it.type
        };
    }
    return d;
}

// только английские значения, как просили: Emergency | Standard | Normal
static string? ReadItsmLevel(string json)
{
    var lvl = TryRead<string>(json, "processData", "itsmLevel")
           ?? TryRead<string>(json, "processData", "level")
           ?? TryRead<string>(json, "processData", "priority");

    if (string.IsNullOrWhiteSpace(lvl)) return null;

    var v = lvl.Trim();
    return v.Equals("Emergency", StringComparison.OrdinalIgnoreCase) ? "Emergency" :
           v.Equals("Standard",  StringComparison.OrdinalIgnoreCase) ? "Standard"  :
           v.Equals("Normal",    StringComparison.OrdinalIgnoreCase) ? "Normal"    :
           null;
}





var documentTitle = TryRead<string>(payloadJson, "processData", "documentTitle");


private static T? TryRead<T>(string json, params string[] path)
{
    try
    {
        JsonNode? node = JsonNode.Parse(json);
        foreach (var p in path)
        {
            if (node is not JsonObject obj) return default;
            node = obj.FirstOrDefault(kv => 
                    string.Equals(kv.Key, p, StringComparison.OrdinalIgnoreCase)).Value;
            if (node is null) return default;
        }
        return node.Deserialize<T>(new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
    }
    catch { return default; }
}




if (string.IsNullOrWhiteSpace(TryRead<string>(payloadJson, "regData", "startdate")))
{
    payloadJson = SetInPayload(payloadJson, jsonOptions, ("regData", "startdate", DateTime.Now.ToString("o")));
}
