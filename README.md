private static Dictionary<string, object> BuildItsmLevelVariables(Dictionary<string, object>? payload)
{
    if (payload is null) return new();

    static string? ReadStr(Dictionary<string, object> p, string key)
    {
        if (!p.TryGetValue(key, out var v) || v is null) return null;
        var json = JsonSerializer.Serialize(v);
        return JsonSerializer.Deserialize<string>(json, new JsonSerializerOptions
        {
            PropertyNameCaseInsensitive = true
        });
    }

    // поддерживаем разные ключи и опечатку
    string? level =
        ReadStr(payload, "itsmLevel") ??
        ReadStr(payload, "level") ??
        ReadStr(payload, "priority") ??
        ReadStr(payload, "classification") ??
        ReadStr(payload, "classifacation"); // опечатка с фронта

    if (string.IsNullOrWhiteSpace(level)) return new();

    var n = level.Trim().ToLowerInvariant();

    // только английские значения
    string? canonical = n switch
    {
        "emergency" => "Emergency",
        "standard"  => "Standard",
        "normal"    => "Normal",
        _ => null
    };

    return canonical is null
        ? new()
        : new Dictionary<string, object> { ["itsmLevel"] = canonical };
}
