public async Task<BaseResponseDto<StartProcessResponse>> Handle(
    StartProcessCommand command,
    CancellationToken cancellationToken)
{
    // --- 1) Ваша старая логика без изменений ---
    var process = await unitOfWork.ProcessRepository
        .GetByFilterAsync(cancellationToken, p => p.ProcessCode == command.ProcessCode)
        ?? throw new HandlerException(
            $"Процесс с кодом {command.ProcessCode} не найден",
            ErrorCodesEnum.Business);

    var requestNumber = await helperService
        .GenerateRequestNumberAsync(command.ProcessCode, cancellationToken);

    var options = new JsonSerializerOptions
    {
        Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping
    };

    // сериализуем входной payload в строку
    var payloadJson = JsonSerializer.Serialize(command.Payload, options);

    var regData        = payloadReader.ReadSection<RegData>(command.Payload, "regData");
    var processDataDto = payloadReader.ReadSection<ProcessDataDto>(command.Payload, "processData");

    var response = await camundaService.CamundaStartProcess(
        new CamundaStartProcessRequest { processCode = command.ProcessCode });

    var processDataCreatedEvent = new ProcessDataCreatedEvent
    {
        ProcessId         = process.Id,
        ProcessCode       = process.ProcessCode,
        ProcessName       = process.ProcessName,
        RegNumber         = requestNumber,
        InitiatorCode     = regData.UserCode,
        InitiatorName     = regData.UserName,
        StatusCode        = "Started",
        StatusName        = "В работе",
        PayloadJson       = payloadJson,
        Title             = processDataDto.DocumentTitle,
        ProcessInstanceId = response
    };
    await unitOfWork.ProcessDataRepository
        .RaiseEvent(processDataCreatedEvent, cancellationToken);

    // --- 2) NEW: Парсим из готовой строки payloadJson массив files вручную ---
    var root = JsonNode.Parse(payloadJson)!.AsObject();
    var filesArray = root["files"] as JsonArray;

    if (filesArray != null)
    {
        foreach (var node in filesArray.OfType<JsonObject>())
        {
            // извлекаем fileName и fileType
            var fileName = node["fileName"]?.GetValue<string>();
            var fileType = node["fileType"]?.GetValue<string>();

            if (string.IsNullOrWhiteSpace(fileName) || string.IsNullOrWhiteSpace(fileType))
            {
                throw new HandlerException(
                    "Каждый объект в \"files\" должен иметь непустые fileName и fileType",
                    ErrorCodesEnum.Business);
            }

            var fileEvent = new ProcessFileCreatedEvent
            {
                EntityId      = Guid.NewGuid(),                                // PK → Id записи
                ProcessDataId = processDataCreatedEvent.EntityId,              // FK на ProcessData
                FileName      = fileName,
                FileType      = fileType
            };
            await unitOfWork.ProcessFileRepository
                .RaiseEvent(fileEvent, cancellationToken);
        }
        // после всех RaiseEvent можно один раз закоммитить,
        // но JournaledGenericRepository.RaiseEvent по умолчанию сразу коммитит
    }
    // --------------------------------------------------------------------

    // --- 3) Старая логика истории задачи и возвращение ответа ---
    var processTaskHistoryCreatedEvent = new ProcessTaskHistoryCreatedEvent
    {
        ProcessDataId    = processDataCreatedEvent.EntityId,
        TaskId           = processDataCreatedEvent.EntityId,
        Action           = ProcessAction.Start.ToString(),
        BlockName        = "Регистрационная форма",
        Timestamp        = DateTime.Now,
        PayloadJson      = payloadJson,
        Comment          = "",
        Description      = "",
        ProcessCode      = command.ProcessCode,
        ProcessName      = process.ProcessName,
        RegNumber        = requestNumber,
        InitiatorCode    = regData.UserCode,
        InitiatorName    = regData.UserName,
        Title            = processDataDto.DocumentTitle
    };
    await unitOfWork.ProcessTaskHistoryRepository
        .RaiseEvent(processTaskHistoryCreatedEvent, cancellationToken);

    return new BaseResponseDto<StartProcessResponse>
    {
        Data = new StartProcessResponse
        {
            ProcessGuid = processDataCreatedEvent.EntityId,
            RegNumber   = requestNumber
        }
    };
}
