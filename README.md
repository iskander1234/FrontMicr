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
            // --- (1) Старая логика: находим процесс и генерируем номер ---
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

            // сериализуем входной payload
            var payloadJson = JsonSerializer.Serialize(command.Payload, options);

            var regData        = payloadReader.ReadSection<RegData>(command.Payload, "regData");
            var processDataDto = payloadReader.ReadSection<ProcessDataDto>(command.Payload, "processData");

            var response = await camundaService.CamundaStartProcess(
               new CamundaStartProcessRequest { processCode = command.ProcessCode });

            // создаём событие создания ProcessData
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

            // --- (2) NEW: сохраняем привязанные файлы в таблицу ProcessFile ---
            // читаем из payload секцию "files" как список DTO
            var filesSection = payloadReader
                .ReadSection<List<FileDto>>(command.Payload, "files")
                ?? new List<FileDto>();

            foreach (var file in filesSection)
            {
                // валидация — fileName и fileType не должны быть пустыми
                if (string.IsNullOrWhiteSpace(file.FileName)
                 || string.IsNullOrWhiteSpace(file.FileType))
                {
                    throw new HandlerException(
                        "Каждый элемент в секции files должен иметь fileName и fileType",
                        ErrorCodesEnum.Business);
                }

                var fileEvent = new ProcessFileCreatedEvent
                {
                    EntityId      = Guid.NewGuid(),                                // Id записи
                    ProcessDataId = processDataCreatedEvent.EntityId,              // FK → только что созданный ProcessData
                    FileName      = file.FileName,                                 // имя файла из DTO
                    FileType      = file.FileType                                  // тип/расширение из DTO
                };
                await unitOfWork.ProcessFileRepository
                    .RaiseEvent(fileEvent, cancellationToken);
            }
            // ------------------------------------------------------------------

            // --- (3) Старая логика: логируем первую задачу и возвращаем ответ ---
            var processTaskHistoryCreatedEvent = new ProcessFileHistoryCreatedEvent
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
    }
