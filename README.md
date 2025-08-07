 public async Task<BaseResponseDto<StartProcessResponse>> Handle(StartProcessCommand command, CancellationToken cancellationToken)
        {
            var process = await unitOfWork.ProcessRepository.GetByFilterAsync(cancellationToken, p => p.ProcessCode == command.ProcessCode)
                ?? throw new HandlerException($"Процесс с кодом {command.ProcessCode} не найден", ErrorCodesEnum.Business);

            var requestNumber = await helperService.GenerateRequestNumberAsync(command.ProcessCode, cancellationToken);

            var options = new JsonSerializerOptions
            {
                Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping
            };

            var payloadJson = JsonSerializer.Serialize(command.Payload, options);
            var regData = payloadReader.ReadSection<RegData>(command.Payload, "regData");
            var processDataDto = payloadReader.ReadSection<ProcessDataDto>(command.Payload, "processData");

            var response = await camundaService.CamundaStartProcess(
               new CamundaStartProcessRequest
               {
                   processCode = command.ProcessCode
               });

            var processDataCreatedEvent = new ProcessDataCreatedEvent
            {
                ProcessId = process.Id,
                ProcessCode = process.ProcessCode,
                ProcessName = process.ProcessName,
                RegNumber = requestNumber,
                InitiatorCode = regData.UserCode,
                InitiatorName = regData.UserName,
                StatusCode = "Started",
                StatusName = "В работе",
                PayloadJson = payloadJson,
                Title = processDataDto.DocumentTitle,
                ProcessInstanceId = response
            };

            await unitOfWork.ProcessDataRepository.RaiseEvent(processDataCreatedEvent, cancellationToken);

            var processTaskHistoryCreatedEvent = new ProcessTaskHistoryCreatedEvent
            {
                ProcessDataId = processDataCreatedEvent.EntityId,
                TaskId = processDataCreatedEvent.EntityId, // Возможно стоит заменить на ID задачи, если будет доступен
                Action = ProcessAction.Start.ToString(),
                BlockName = "Регистрационная форма",
                Timestamp = DateTime.Now,
                PayloadJson = payloadJson,
                Comment = "",
                Description = "",
                ProcessCode = command.ProcessCode,
                ProcessName = process.ProcessName,
                RegNumber = requestNumber,
                InitiatorCode = regData.UserCode,
                InitiatorName = regData.UserName,
                Title = processDataDto.DocumentTitle
            };

            await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(processTaskHistoryCreatedEvent, cancellationToken);

            
            return new BaseResponseDto<StartProcessResponse>
            {
                Data = new StartProcessResponse
                {
                    ProcessGuid = processDataCreatedEvent.EntityId,
                    RegNumber = requestNumber
                }
            };
        }
