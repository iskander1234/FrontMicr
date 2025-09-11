// ★ Rework: если отправляем дальше (Submit) и в JSON уже есть regData.regnum — делаем полный update ProcessData
if (Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var currentStage)
    && currentStage == ProcessStage.Rework
    && command.Action == ProcessAction.Submit)
{
    // command.PayloadJson у тебя Dictionary<string, object>? — если пришёл новый payload, сериализуем его.
    string candidateJson = command.PayloadJson is not null
        ? JsonSerializer.Serialize(command.PayloadJson)
        : processData.PayloadJson;

    if (!string.IsNullOrWhiteSpace(candidateJson) && JsonHasRegnum(candidateJson))
    {
        // Сохраняем ВСЁ (как "простой update") через событийную модель
        await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataEditedEvent
        {
            EntityId      = processData.Id,
            StatusCode    = processData.StatusCode,
            StatusName    = processData.StatusName,
            PayloadJson   = candidateJson,               // если пришёл новый — сохранится новый
            InitiatorCode = processData.InitiatorCode,
            InitiatorName = processData.InitiatorName,
            Title         = processData.Title
        }, cancellationToken);
        // дальше вся твоя логика идёт как была
    }
}


private static bool JsonHasRegnum(string json)
{
    try
    {
        using var doc = JsonDocument.Parse(json);
        var root = doc.RootElement;
        if (root.ValueKind != JsonValueKind.Object) return false;

        // case-insensitive поиск regData.regnum
        foreach (var p in root.EnumerateObject())
        {
            if (!p.Name.Equals("regData", StringComparison.OrdinalIgnoreCase)) continue;
            var regData = p.Value;
            if (regData.ValueKind != JsonValueKind.Object) return false;

            foreach (var rp in regData.EnumerateObject())
            {
                if (!rp.Name.Equals("regnum", StringComparison.OrdinalIgnoreCase)) continue;
                return rp.Value.ValueKind == JsonValueKind.String && !string.IsNullOrWhiteSpace(rp.Value.GetString());
            }
        }
    }
    catch
    {
        // битый JSON — считаем, что regnum нет
    }
    return false;
}
