public async Task<BaseResponseDto<StartProcessResponse>> Handle(
    StartProcessCommand command,
    CancellationToken cancellationToken)
{
    // 1. Находим процесс по коду
    var process = await unitOfWork.ProcessRepository
        .GetByFilterAsync(cancellationToken, p => p.ProcessCode == command.ProcessCode)
        ?? throw new HandlerException(
            $"Процесс с кодом {command.ProcessCode} не найден",
            ErrorCodesEnum.Business);

    // 2. Генерируем регистрационный номер
    var requestNumber = await helperService
        .GenerateRequestNumberAsync(command.ProcessCode, cancellationToken);

    // 3. Параметры сериализации JSON
    var options = new JsonSerializerOptions
    {
        Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping
    };

    // 4. Сериализуем полный payload
    var payloadJson = JsonSerializer.Serialize(command.Payload, options);

    // 5. Читаем из payload секции regData и processData
    var regData        = payloadReader.ReadSection<RegData>(command.Payload, "regData");
    var processDataDto = payloadReader.ReadSection<ProcessDataDto>(command.Payload, "processData");

    // 6. Стартуем процесс в Камунде
    var response = await camundaService.CamundaStartProcess(
        new CamundaStartProcessRequest { processCode = command.ProcessCode });

    // 7. Создаём событие создания ProcessData
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

    // --- NEW: сохраняем привязанные файлы в таблицу ProcessFile ---
    // Предполагаем, что в command.Payload есть список файлов:
    var filesSection = payloadReader.ReadSection<List<FileDto>>(command.Payload, "files");
    if (filesSection != null && filesSection.Any())
    {
        foreach (var file in filesSection)
        {
            var fileEvent = new ProcessFileCreatedEvent
            {
                EntityId      = Guid.NewGuid(),                               // PK → Id
                ProcessDataId = processDataCreatedEvent.EntityId,             // FK на только что созданный ProcessData
                FileName      = file.FileName,                                // имя файла из DTO
                FileType      = file.FileType                                 // тип/расширение из DTO
            };
            await unitOfWork.ProcessFileRepository
                .RaiseEvent(fileEvent, cancellationToken);
        }
    }
    // ------------------------------------------------------------------

    // 8. Логируем историю первой задачи
    var processTaskHistoryCreatedEvent = new ProcessTaskHistoryCreatedEvent
    {
        ProcessDataId    = processDataCreatedEvent.EntityId,
        TaskId           = processDataCreatedEvent.EntityId, // при старте процесса пока нет отдельной задачи
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

    // 9. Возвращаем ответ
    return new BaseResponseDto<StartProcessResponse>
    {
        Data = new StartProcessResponse
        {
            ProcessGuid = processDataCreatedEvent.EntityId,
            RegNumber   = requestNumber
        }
    };
}
