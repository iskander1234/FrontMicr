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

    // ★ синхронизируем таблицу файлов под новый payload
    await ReconcileFilesAsync(processData.Id, mergedJson, unitOfWork, cancellationToken);   // ← ДОБАВЬ ЭТУ СТРОКУ

    // держим актуальную версию локально, чтобы ниже читалась новая
    processData.PayloadJson = mergedJson;
}


private static async Task ReconcileFilesAsync(
    Guid processDataId,
    string payloadJson,
    IUnitOfWork unitOfWork,
    CancellationToken ct)
{
    // 1) Достаём массив files из payload (если его нет — выходим)
    JsonArray? filesArr = null;
    try
    {
        var root = JsonNode.Parse(payloadJson)!.AsObject();
        filesArr = root["files"] as JsonArray;
    }
    catch { /* игнор, считаем что files нет */ }

    if (filesArr is null)
        return;

    // 2) Готовим целевой набор из payload
    var incoming = new Dictionary<Guid, (string fileName, string fileType)>();
    foreach (var node in filesArr.OfType<JsonObject>())
    {
        var fileIdStr = node["fileId"]?.GetValue<string>();
        var fileName  = node["fileName"]?.GetValue<string>();
        var fileType  = node["fileType"]?.GetValue<string>();

        if (string.IsNullOrWhiteSpace(fileIdStr) || !Guid.TryParse(fileIdStr, out var fileId))
            throw new HandlerException("Каждый объект в \"files\" должен иметь корректный GUID в поле fileId", ErrorCodesEnum.Business);
        if (string.IsNullOrWhiteSpace(fileName) || string.IsNullOrWhiteSpace(fileType))
            throw new HandlerException("Каждый объект в \"files\" должен иметь непустые fileName и fileType", ErrorCodesEnum.Business);

        // дубликаты в payload перетрутся последним значением
        incoming[fileId] = (fileName!, fileType!);
    }

    // 3) Текущее состояние БД
    var existing = await unitOfWork.ProcessFileRepository.GetByFilterListAsync(ct, f => f.ProcessDataId == processDataId);
    var existingById = existing
        .Where(e => e.FileId != Guid.Empty)
        .ToDictionary(e => e.FileId, e => e);

    var incomingIds = incoming.Keys.ToHashSet();

    // 4) ADD: создать недостающие
    foreach (var kv in incoming)
    {
        var fileId = kv.Key;
        var (fileName, fileType) = kv.Value;

        if (!existingById.ContainsKey(fileId))
        {
            await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileCreatedEvent
            {
                EntityId     = Guid.NewGuid(),
                ProcessDataId= processDataId,
                FileId       = fileId,
                FileName     = fileName,
                FileType     = fileType
            }, ct);
        }
        else
        {
            // (опционально) если имя/тип поменялись — тут можно поднять Update-событие, если оно у тебя есть
            // пример:
            // if (existingById[fileId].FileName != fileName || existingById[fileId].FileType != fileType)
            //     await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileUpdatedEvent { ... }, ct);
        }
    }

    // 5) DELETE: убрать те, которых нет в payload
    foreach (var ex in existingById.Values)
    {
        if (!incomingIds.Contains(ex.FileId))
        {
            // Замени на реальное событие удаления в твоём домене, если имя другое
            await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileDeletedEvent
            {
                EntityId = ex.Id
            }, ct);
        }
    }
}
