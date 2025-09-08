// ... using-и и namespace как у тебя ...

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
        var process = await unitOfWork.ProcessRepository
                          .GetByFilterAsync(cancellationToken, p => p.ProcessCode == command.ProcessCode)
                      ?? throw new HandlerException($"Процесс с кодом {command.ProcessCode} не найден",
                          ErrorCodesEnum.Business);

        var jsonOptions = new JsonSerializerOptions { Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping };
        var payloadJson = JsonSerializer.Serialize(command.Payload, jsonOptions);

        var regData        = payloadReader.ReadSection<RegData>(command.Payload, "regData");
        var processDataDto = payloadReader.ReadSection<ProcessDataDto>(command.Payload, "processData");

        // NEW: читаем номер из payload.regData.regnum
        var regnumFromPayload = TryGetRegnumFromPayload(payloadJson);

        // === A) если regnum задан, пробуем использовать существующую заявку ===
        if (!string.IsNullOrWhiteSpace(regnumFromPayload))
        {
            var alreadyStarted = await unitOfWork.ProcessDataRepository.GetByFilterAsync(
                cancellationToken,
                p => p.ProcessCode == process.ProcessCode
                     && p.RegNumber == regnumFromPayload
                     && p.StatusCode == "Started"
            );
            if (alreadyStarted is not null)
            {
                return new BaseResponseDto<StartProcessResponse>
                {
                    Data = new StartProcessResponse
                        { ProcessGuid = alreadyStarted.Id, RegNumber = alreadyStarted.RegNumber }
                };
            }

            var draft = await unitOfWork.ProcessDataRepository.GetByFilterAsync(
                cancellationToken,
                p => p.ProcessCode == process.ProcessCode
                     && p.RegNumber == regnumFromPayload
                     && p.StatusCode == "Draft"
            );

            if (draft is not null)
            {
                // NEW: синхронизируем payload.regData.regnum с реальным номером черновика
                payloadJson = SetRegnumInPayload(payloadJson, draft.RegNumber, jsonOptions);

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
                await AddFilesUpsertAsync(draft.Id, payloadJson, cancellationToken);
                await WriteHistoryStartIfAbsentAsync(draft.Id, draft.RegNumber, payloadJson, cancellationToken);

                return new BaseResponseDto<StartProcessResponse>
                {
                    Data = new StartProcessResponse { ProcessGuid = draft.Id, RegNumber = draft.RegNumber }
                };
            }
        }

        // === B) черновика нет — создаём новую Started ===
        var requestNumber = await helperService.GenerateRequestNumberAsync(command.ProcessCode, cancellationToken);

        // NEW: вписываем номер в payload.regData.regnum
        payloadJson = SetRegnumInPayload(payloadJson, requestNumber, jsonOptions);

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

        // ---------- ЛОКАЛЬНЫЕ ФУНКЦИИ (NEW) ----------

        static string? TryGetRegnumFromPayload(string payload)
        {
            try
            {
                var root = JsonNode.Parse(payload)!.AsObject();
                return root["regData"]?["regnum"]?.GetValue<string>()?.Trim();
            }
            catch { return null; }
        }

        static string SetRegnumInPayload(string payload, string regnum, JsonSerializerOptions opts)
        {
            var root = JsonNode.Parse(payload)!.AsObject();
            if (root["regData"] is not JsonObject rd)
            {
                rd = new JsonObject();
                root["regData"] = rd;
            }
            rd["regnum"] = regnum;
            return JsonSerializer.Serialize(root, opts);
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

                var existing = await unitOfWork.ProcessFileRepository.GetByFilterListAsync(
                    ct, f => f.ProcessDataId == processDataId);
                var existingIds = existing.Where(x => x.FileId != Guid.Empty)
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

                    if (existingIds.Contains(fileId)) continue;

                    await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileCreatedEvent
                    {
                        EntityId      = Guid.NewGuid(),
                        ProcessDataId = processDataId,
                        FileId        = fileId,
                        FileName      = fileName,
                        FileType      = fileType
                    }, ct);

                    existingIds.Add(fileId);
                }
            }
            catch { /* опционально залогировать */ }
        }

        async Task WriteHistoryStartIfAbsentAsync(Guid processDataId, string regNumber, string payload, CancellationToken ct)
        {
            var hasStart = await unitOfWork.ProcessTaskHistoryRepository.CountAsync(
                ct, h => h.ProcessDataId == processDataId && h.Action == ProcessAction.Start.ToString());
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






// ... using-и и namespace как у тебя ...

public class SaveProcessCommandHandler(
    IUnitOfWork unitOfWork,
    IPayloadReaderService payloadReader,
    IProcessTaskService helperService)
    : IRequestHandler<SaveProcessCommand, BaseResponseDto<StartProcessResponse>>
{
    public async Task<BaseResponseDto<StartProcessResponse>> Handle(
        SaveProcessCommand command,
        CancellationToken cancellationToken)
    {
        var process = await unitOfWork.ProcessRepository
            .GetByFilterAsync(cancellationToken, p => p.ProcessCode == command.ProcessCode)
            ?? throw new HandlerException(
                $"Процесс с кодом {command.ProcessCode} не найден",
                ErrorCodesEnum.Business);

        var requestNumber = await helperService
            .GenerateRequestNumberAsync(command.ProcessCode, cancellationToken);

        var options    = new JsonSerializerOptions { Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping };
        var payloadJson = JsonSerializer.Serialize(command.Payload, options);

        var regData        = payloadReader.ReadSection<RegData>(command.Payload, "regData");
        var processDataDto = payloadReader.ReadSection<ProcessDataDto>(command.Payload, "processData");

        // NEW: вписываем номер в payload.regData.regnum
        payloadJson = SetRegnumInPayload(payloadJson, requestNumber, options);

        var processDataCreatedEvent = new ProcessDataCreatedEvent
        {
            ProcessId     = process.Id,
            ProcessCode   = process.ProcessCode,
            ProcessName   = process.ProcessName,
            RegNumber     = requestNumber,
            InitiatorCode = regData.UserCode,
            InitiatorName = regData.UserName,
            StatusCode    = "Draft",
            StatusName    = "Черновик",
            PayloadJson   = payloadJson,
            Title         = processDataDto.DocumentTitle
        };

        await unitOfWork.ProcessDataRepository
            .RaiseEvent(processDataCreatedEvent, cancellationToken);

        return new BaseResponseDto<StartProcessResponse>
        {
            Data = new StartProcessResponse
            {
                ProcessGuid = processDataCreatedEvent.EntityId,
                RegNumber   = requestNumber
            }
        };

        // helper (NEW)
        static string SetRegnumInPayload(string payload, string regnum, JsonSerializerOptions opts)
        {
            var root = JsonNode.Parse(payload)!.AsObject();
            if (root["regData"] is not JsonObject rd)
            {
                rd = new JsonObject();
                root["regData"] = rd;
            }
            rd["regnum"] = regnum;
            return JsonSerializer.Serialize(root, opts);
        }
    }
}








