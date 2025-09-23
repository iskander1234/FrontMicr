// было
var variables = new Dictionary<string, object>
{
    ["classification"] = classification
};

// ДОБАВКА: Логика для Level1/Level2/Level3
if (Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var stage))
{
    switch (stage)
    {
        case ProcessStage.Level1:
        {
            // "Line 1" определяется по execution: executed => true, toSecondLine => false
            var execution = ReadExecution(command.PayloadJson); // "executed" | "toSecondLine" | null
            bool line1 = string.Equals(execution, "executed", StringComparison.OrdinalIgnoreCase)
                      ? true
                      : string.Equals(execution, "toSecondLine", StringComparison.OrdinalIgnoreCase)
                        ? false
                        : true; // дефолт, если нет execution
            variables["Line 1"] = line1;
            break;
        }
        case ProcessStage.Level2:
        {
            bool line2 = ReadLineBool(command.PayloadJson, 2) ?? true;
            variables["Line 2"] = line2;
            break;
        }
        case ProcessStage.Level3:
        {
            bool line3 = ReadLineBool(command.PayloadJson, 3) ?? true;
            variables["Line 3"] = line3;
            break;
        }
    }
}
// КОНЕЦ ДОБАВКИ

// дальше как у тебя:
var submit = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
{
    TaskId    = claimed.TaskId,
    Variables = variables
});










private static string? ReadExecution(Dictionary<string, object>? payload)
{
    // ищем: payload -> processData -> level1 -> formData -> execution
    try
    {
        if (payload is null) return null;

        if (!payload.TryGetValue("payload", out var pRaw) || pRaw is null) return null;
        var p = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pRaw));

        if (p is null || !p.TryGetValue("processData", out var pdRaw) || pdRaw is null) return null;
        var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pdRaw));

        if (pd is null || !pd.TryGetValue("level1", out var lRaw) || lRaw is null) return null;
        var level1 = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(lRaw));

        if (level1 is null || !level1.TryGetValue("formData", out var fdRaw) || fdRaw is null) return null;
        var form = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(fdRaw));

        if (form is null || !form.TryGetValue("execution", out var exRaw) || exRaw is null) return null;
        return JsonSerializer.Deserialize<string>(JsonSerializer.Serialize(exRaw));
    }
    catch { return null; }
}

private static bool? ReadLineBool(Dictionary<string, object>? payload, int levelIndex)
{
    // Level2: payload->processData->level2->formData->line2 (bool)
    // Level3: payload->processData->level3->formData->line3 (bool)
    try
    {
        if (payload is null) return null;

        if (!payload.TryGetValue("payload", out var pRaw) || pRaw is null) return null;
        var p = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pRaw));

        if (p is null || !p.TryGetValue("processData", out var pdRaw) || pdRaw is null) return null;
        var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pdRaw));

        var levelKey = $"level{levelIndex}";
        if (pd is null || !pd.TryGetValue(levelKey, out var lRaw) || lRaw is null) return null;
        var level = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(lRaw));

        if (level is null || !level.TryGetValue("formData", out var fdRaw) || fdRaw is null) return null;
        var form = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(fdRaw));

        var lineKey = $"line{levelIndex}";
        if (form is null || !form.TryGetValue(lineKey, out var lineRaw) || lineRaw is null) return null;

        return JsonSerializer.Deserialize<bool?>(JsonSerializer.Serialize(lineRaw));
    }
    catch { return null; }
}
