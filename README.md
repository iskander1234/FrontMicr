namespace BpmBaseApi.Domain.Entities.Process
{
    public enum ProcessFileState { Active = 0, Removed = 1 }

    public class ProcessFileEntity : BaseJournaledEntity
    {
        public Guid FileId { get; set; }
        public string FileName { get; set; } = default!;
        public string FileType { get; set; } = default!;
        public Guid ProcessDataId { get; set; }
        public ProcessDataEntity ProcessData { get; set; } = default!;

        // NEW: soft-delete признак (0 — активен, 1 — удалён)
        public ProcessFileState State { get; set; } = ProcessFileState.Active;
        public bool IsActive => State == ProcessFileState.Active;

        public ProcessFileEntity() { }

        #region Apply
        public void Apply(ProcessFileCreatedEvent @event)
        {
            Id            = @event.EntityId;
            FileId        = @event.FileId;
            FileName      = @event.FileName;
            FileType      = @event.FileType;
            ProcessDataId = @event.ProcessDataId;
            State         = ProcessFileState.Active; // 0 при создании
        }

        public void Apply(SignDocumentCreateEvent @event)
        {
            Id            = @event.EntityId;
            FileId        = @event.FileId;
            FileName      = @event.FileName;
            FileType      = @event.FileType;
            ProcessDataId = @event.ProcessDataId;
            State         = ProcessFileState.Active;
        }

        // Если где-то ещё поднимается ProcessFileDeletedEvent — трактуем как soft-delete
        public void Apply(ProcessFileDeletedEvent @event)
        {
            State = ProcessFileState.Removed; // 1
        }

        // NEW: универсальное событие переключения состояния/обновления подписи
        public void Apply(ProcessFileStatusChangedEvent @event)
        {
            State = @event.State;
            if (!string.IsNullOrWhiteSpace(@event.FileName)) FileName = @event.FileName!;
            if (!string.IsNullOrWhiteSpace(@event.FileType)) FileType = @event.FileType!;
        }
        #endregion
    }
}



namespace BpmBaseApi.Domain.Entities.Event.Process
{
    public class ProcessFileStatusChangedEvent : BaseDomainEvent
    {
        public Guid EntityId { get; set; }
        public ProcessFileState State { get; set; } // 0/1
        public string? FileName { get; set; }
        public string? FileType { get; set; }
    }
}

Start 
async Task AddFilesUpsertAsync(Guid processDataId, string payload, CancellationToken ct)
{
    try
    {
        var root = JsonNode.Parse(payload)!.AsObject();
        var files = root["files"] as JsonArray;
        if (files is null || files.Count == 0) return;

        // важно: берём все (и удалённые, и активные)
        var existing = await unitOfWork.ProcessFileRepository.GetByFilterListAsync(
            ct, f => f.ProcessDataId == processDataId);

        var byFileId = existing
            .Where(x => x.FileId != Guid.Empty)
            .ToDictionary(x => x.FileId, x => x);

        foreach (var node in files.OfType<JsonObject>())
        {
            var fileIdStr = node["fileId"]?.GetValue<string>();
            var fileName  = node["fileName"]?.GetValue<string>();
            var fileType  = node["fileType"]?.GetValue<string>();

            if (string.IsNullOrWhiteSpace(fileName) || string.IsNullOrWhiteSpace(fileType))
                throw new HandlerException("Каждый объект в \"files\" должен иметь непустые fileName и fileType",
                    ErrorCodesEnum.Business);
            if (string.IsNullOrWhiteSpace(fileIdStr) || !Guid.TryParse(fileIdStr, out var fileId))
                throw new HandlerException("Каждый объект в \"files\" должен иметь корректный GUID в поле fileId",
                    ErrorCodesEnum.Business);

            if (!byFileId.TryGetValue(fileId, out var ex))
            {
                await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileCreatedEvent
                {
                    EntityId      = Guid.NewGuid(),
                    ProcessDataId = processDataId,
                    FileId        = fileId,
                    FileName      = fileName!,
                    FileType      = fileType!
                }, ct);
            }
            else
            {
                // если был удалён раньше — вернём в активные; если имя/тип изменились — обновим
                if (ex.State != ProcessFileState.Active || ex.FileName != fileName || ex.FileType != fileType)
                {
                    await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileStatusChangedEvent
                    {
                        EntityId = ex.Id,
                        State    = ProcessFileState.Active,
                        FileName = fileName,
                        FileType = fileType
                    }, ct);
                }
            }
        }
    }
    catch
    {
        /* логирование по желанию */
    }
}



send 

private static async Task ReconcileFilesAsync(
    Guid processDataId,
    string payloadJson,
    IUnitOfWork unitOfWork,
    CancellationToken ct)
{
    // 1) Собираем итоговый список из payload
    JsonArray? filesArr = null;
    try
    {
        var root = JsonNode.Parse(payloadJson)!.AsObject();
        filesArr = root["files"] as JsonArray;
    }
    catch { /* нет секции files — выходим */ }

    var incoming = new Dictionary<Guid, (string fileName, string fileType)>();
    if (filesArr is not null)
    {
        foreach (var node in filesArr.OfType<JsonObject>())
        {
            var fileIdStr = node["fileId"]?.GetValue<string>();
            var fileName  = node["fileName"]?.GetValue<string>();
            var fileType  = node["fileType"]?.GetValue<string>();

            if (string.IsNullOrWhiteSpace(fileIdStr) || !Guid.TryParse(fileIdStr, out var fileId))
                throw new HandlerException("Каждый объект в \"files\" должен иметь корректный GUID в поле fileId",
                    ErrorCodesEnum.Business);
            if (string.IsNullOrWhiteSpace(fileName) || string.IsNullOrWhiteSpace(fileType))
                throw new HandlerException("Каждый объект в \"files\" должен иметь непустые fileName и fileType",
                    ErrorCodesEnum.Business);

            incoming[fileId] = (fileName!, fileType!);
        }
    }

    // 2) Берём все файлы из БД (и 0 и 1)
    var existing = await unitOfWork.ProcessFileRepository.GetByFilterListAsync(
        ct, f => f.ProcessDataId == processDataId);

    var byFileId = existing
        .Where(e => e.FileId != Guid.Empty)
        .ToDictionary(e => e.FileId, e => e);

    var incomingIds = incoming.Keys.ToHashSet();

    // 3) ADD/UNDELETE/RENAME
    foreach (var kv in incoming)
    {
        var fileId = kv.Key;
        var (fileName, fileType) = kv.Value;

        if (!byFileId.TryGetValue(fileId, out var ex))
        {
            // совсем новый
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
            // был, но мог быть удалён или изменились имя/тип
            if (ex.State != ProcessFileState.Active || ex.FileName != fileName || ex.FileType != fileType)
            {
                await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileStatusChangedEvent
                {
                    EntityId = ex.Id,
                    State    = ProcessFileState.Active,
                    FileName = fileName,
                    FileType = fileType
                }, ct);
            }
        }
    }

    // 4) SOFT DELETE: всё, чего нет в payload → State = Removed (1)
    foreach (var ex in byFileId.Values)
    {
        if (!incomingIds.Contains(ex.FileId) && ex.State != ProcessFileState.Removed)
        {
            // Можно использовать старый ProcessFileDeletedEvent, он теперь ставит State=Removed
            // await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileDeletedEvent { EntityId = ex.Id }, ct);

            // Предпочтительно: единое событие статуса
            await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileStatusChangedEvent
            {
                EntityId = ex.Id,
                State    = ProcessFileState.Removed
            }, ct);
        }
    }
}
