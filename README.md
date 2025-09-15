 await ReconcileFilesAsync(processData.Id, mergedJson, unitOfWork, cancellationToken);


 private static async Task ReconcileFilesAsync(
    Guid processDataId,
    string payloadJson,
    IUnitOfWork unitOfWork,
    CancellationToken ct)
{
    // 1) Считываем итоговый список файлов из payload
    JsonArray? filesArr = null;
    try
    {
        var root = JsonNode.Parse(payloadJson)!.AsObject();
        filesArr = root["files"] as JsonArray;
    }
    catch { /* нет/битый files — молча выходим */ }

    if (filesArr is null) return;

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

        incoming[fileId] = (fileName!, fileType!); // dedupe по fileId
    }

    // 2) Текущее состояние файлов в БД
    var existing = await unitOfWork.ProcessFileRepository.GetByFilterListAsync(
        ct, f => f.ProcessDataId == processDataId);

    var existingById = existing
        .Where(e => e.FileId != Guid.Empty)
        .ToDictionary(e => e.FileId, e => e);

    var incomingIds = incoming.Keys.ToHashSet();

    // 3) ADD: создаём отсутствующие в БД файлы
    foreach (var kv in incoming)
    {
        var fileId = kv.Key;
        var (fileName, fileType) = kv.Value;

        if (!existingById.ContainsKey(fileId))
        {
            await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileCreatedEvent
            {
                EntityId      = Guid.NewGuid(),
                ProcessDataId = processDataId,
                FileId        = fileId,
                FileName      = fileName,
                FileType      = fileType
            }, ct);
        }
        else
        {
            // (опционально) если имя/тип поменялись — подними Update-событие, если оно есть
            // var ex = existingById[fileId];
            // if (ex.FileName != fileName || ex.FileType != fileType)
            //     await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileUpdatedEvent { ... }, ct);
        }
    }

    // 4) DELETE: удаляем те, которых больше нет в итоговом payload
    foreach (var ex in existingById.Values)
    {
        if (!incomingIds.Contains(ex.FileId))
        {
            await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileDeletedEvent
            {
                EntityId = ex.Id,
                // ProcessDataId = processDataId // если событие его поддерживает — добавь
            }, ct);
        }
    }
}
