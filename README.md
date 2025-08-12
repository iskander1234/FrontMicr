private static bool IsSequential(Dictionary<string, object>? payload)
{
    if (payload == null) return false;
    if (!payload.TryGetValue("sysInfo", out var sysRaw) || sysRaw is null) return false;

    var json = JsonSerializer.Serialize(sysRaw);
    var sys  = JsonSerializer.Deserialize<Dictionary<string, object>>(json);

    if (sys != null && sys.TryGetValue("sequence", out var v) && v is not null)
    {
        if (v is bool b) return b;
        if (bool.TryParse(v.ToString(), out var parsed)) return parsed;
    }
    return false;
}
