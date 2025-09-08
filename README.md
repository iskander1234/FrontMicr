public class StartProcessCommandHandler(
    IUnitOfWork unitOfWork,
    IPayloadReaderService payloadReader,
    IProcessTaskService helperService,
    ICamundaService camundaService)
    : IRequestHandler<StartProcessCommand, BaseResponseDto<StartProcessResponse>>
{
    public async Task<BaseResponseDto<StartProcessResponse>> Handle(
        StartProcessCommand command,
        CancellationToken cancellationToken)
    {
        // 1) Процесс
        var process = await unitOfWork.ProcessRepository
            .GetByFilterAsync(cancellationToken, p => p.ProcessCode == command.ProcessCode)
            ?? throw new HandlerException($"Процесс с кодом {command.ProcessCode} не найден", ErrorCodesEnum.Business);

        // 2) Готовим payload
        var jsonOptions = new JsonSerializerOptions { Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping };
        var payloadJson = JsonSerializer.Serialize(command.Payload, jsonOptions);

        var regData        = payloadReader.ReadSection<RegData>(command.Payload, "regData");
        var processDataDto = payloadReader.ReadSection<ProcessDataDto>(command.Payload, "processData");
        var regNumberFromPayload = TryGetRegNumberFromPayload(payloadJson);

        // === A) Если в payload есть regNumber, пробуем использовать существующую заявку ===
        if (!string.IsNullOrWhiteSpace(regNumberFromPayload))
        {
            // 1) Если уже есть Started с этим regNumber — просто вернём его (idempotency)
            var alreadyStarted = await unitOfWork.ProcessDataRepository.GetByFilterAsync(
                cancellationToken,
                p => p.ProcessCode == process.ProcessCode
                     && p.RegNumber == regNumberFromPayload
                     && p.StatusCode == "Started"
            );
            if (alreadyStarted is not null)
            {
                // можно (при желании) обновить payload/титул, но чаще лучше ничего не трогать
                return new BaseResponseDto<StartProcessResponse>
                {
                    Data = new StartProcessResponse { ProcessGuid = alreadyStarted.Id, RegNumber = alreadyStarted.RegNumber }
                };
            }

            // 2) Есть Draft — апдейтим его и стартуем
            var draft = await unitOfWork.ProcessDataRepository.GetByFilterAsync(
                cancellationToken,
                p => p.ProcessCode == process.ProcessCode
                     && p.RegNumber == regNumberFromPayload
                     && p.StatusCode == "Draft"
            );

            if (draft is not null)
            {
                await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataEditedEvent
                {
                    EntityId      = draft.Id,
                    StatusCode    = "Started",
                    StatusName    = "В работе",
                    PayloadJson   = payloadJson,
                    InitiatorCode = regData?.UserCode,
                    InitiatorName = regData?.UserName,
                    Title         = processDataDto?.DocumentTitle ?? draft.Title
                }, cancellationToken);

                await StartCamundaAndLinkAsync(draft.Id, cancellationToken);

                // files: upsert (только новые fileId)
                await AddFilesUpsertAsync(draft.Id, payloadJson, cancellationToken);

                // history: записываем Start только если его ещё нет
                await WriteHistoryStartIfAbsentAsync(draft.Id, draft.RegNumber, payloadJson, cancellationToken);

                return new BaseResponseDto<StartProcessResponse>
                {
                    Data = new StartProcessResponse { ProcessGuid = draft.Id, RegNumber = draft.RegNumber }
                };
            }
        }

        // === B) Черновика нет — создаём НОВУЮ заявку и стартуем (как раньше) ===
        // requestNumber генерим ТОЛЬКО здесь, чтобы не плодить «лишние» номера.
        var requestNumber = await helperService.GenerateRequestNumberAsync(command.ProcessCode, cancellationToken);

        var createdEvt = new ProcessDataCreatedEvent
        {
            ProcessId     = process.Id,
            ProcessCode   = process.ProcessCode,
            ProcessName   = process.ProcessName,
            RegNumber     = requestNumber,
            InitiatorCode = regData.UserCode,
            InitiatorName = regData.UserName,
            StatusCode    = "Started",
            StatusName    = "В работе",
            PayloadJson   = payloadJson,
            Title         = processDataDto.DocumentTitle
        };
        await unitOfWork.ProcessDataRepository.RaiseEvent(createdEvt, cancellationToken);

        await StartCamundaAndLinkAsync(createdEvt.EntityId, cancellationToken);
        await AddFilesUpsertAsync(createdEvt.EntityId, payloadJson, cancellationToken);
        await WriteHistoryStartIfAbsentAsync(createdEvt.EntityId, requestNumber, payloadJson, cancellationToken);

        return new BaseResponseDto<StartProcessResponse>
        {
            Data = new StartProcessResponse { ProcessGuid = createdEvt.EntityId, RegNumber = requestNumber }
        };

        // ---------- локальные функции ----------

        static string? TryGetRegNumberFromPayload(string payload)
        {
            try
            {
                var root = JsonNode.Parse(payload)!.AsObject();
                return root["regData"]?["regNumber"]?.GetValue<string>();
            }
            catch { return null; }
        }

        async Task StartCamundaAndLinkAsync(Guid processDataId, CancellationToken ct)
        {
            var pi = await camundaService.CamundaStartProcess(new CamundaStartProcessRequest
            {
                processCode = command.ProcessCode,
                variables   = new Dictionary<string, object> { { "processGuid", processDataId.ToString() } }
            });

            await unitOfWork.ProcessDataRepository.RaiseEvent(
                new ProcessDataProcessInstanseIdChangedEvent
                {
                    EntityId          = processDataId,
                    ProcessInstanceId = pi
                },
                ct);
        }

        async Task AddFilesUpsertAsync(Guid processDataId, string payload, CancellationToken ct)
        {
            try
            {
                var root  = JsonNode.Parse(payload)!.AsObject();
                var files = root["files"] as JsonArray;
                if (files is null || files.Count == 0) return;

                // вытащим уже существующие fileIds для этой заявки
                var existing = await unitOfWork.ProcessFileRepository.GetByFilterListAsync(
                    ct,
                    f => f.ProcessDataId == processDataId
                );
                var existingIds = existing
                    .Where(x => x.FileId != Guid.Empty)
                    .Select(x => x.FileId)
                    .ToHashSet();

                foreach (var node in files.OfType<JsonObject>())
                {
                    var fileIdStr = node["fileId"]?.GetValue<string>();
                    var fileName  = node["fileName"]?.GetValue<string>();
                    var fileType  = node["fileType"]?.GetValue<string>();

                    if (string.IsNullOrWhiteSpace(fileName) || string.IsNullOrWhiteSpace(fileType))
                        throw new HandlerException("Каждый объект в \"files\" должен иметь непустые fileName и fileType", ErrorCodesEnum.Business);
                    if (string.IsNullOrWhiteSpace(fileIdStr) || !Guid.TryParse(fileIdStr, out var fileId))
                        throw new HandlerException("Каждый объект в \"files\" должен иметь корректный GUID в поле fileId", ErrorCodesEnum.Business);

                    // добавляем только новые fileId
                    if (existingIds.Contains(fileId)) continue;

                    await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileCreatedEvent
                    {
                        EntityId      = Guid.NewGuid(),
                        ProcessDataId = processDataId,
                        FileId        = fileId,
                        FileName      = fileName,
                        FileType      = fileType
                    }, ct);

                    existingIds.Add(fileId); // чтобы не вставить дважды из одного payload
                }
            }
            catch
            {
                // опционально: лог
            }
        }

        async Task WriteHistoryStartIfAbsentAsync(Guid processDataId, string regNumber, string payload, CancellationToken ct)
        {
            // если уже есть запись Start — не дублируем
            var hasStart = await unitOfWork.ProcessTaskHistoryRepository.CountAsync(
                ct,
                h => h.ProcessDataId == processDataId
                     && h.Action == ProcessAction.Start.ToString()
            );
            if (hasStart > 0) return;

            await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(new ProcessTaskHistoryCreatedEvent
            {
                ProcessDataId = processDataId,
                TaskId        = processDataId,
                Action        = ProcessAction.Start.ToString(),
                BlockName     = "Регистрационная форма",
                Timestamp     = DateTime.Now,
                PayloadJson   = payload,
                Comment       = "",
                Description   = "",
                ProcessCode   = command.ProcessCode,
                ProcessName   = process.ProcessName,
                RegNumber     = regNumber,
                InitiatorCode = regData.UserCode,
                InitiatorName = regData.UserName,
                Title         = processDataDto.DocumentTitle
            }, ct);
        }
    }
}
