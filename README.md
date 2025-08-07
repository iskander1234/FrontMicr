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
