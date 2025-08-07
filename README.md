using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Enum;
using BpmBaseApi.Shared.Models.Camunda;
using BpmBaseApi.Shared.Models.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using System.Text.Encodings.Web;
using System.Text.Json;
using static BpmBaseApi.Shared.Models.Process.CommonData;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    public class StartProcessCommandHandler(
    IUnitOfWork unitOfWork,
    IPayloadReaderService payloadReader,
    IProcessTaskService helperService,
    ICamundaService camundaService)
    : IRequestHandler<StartProcessCommand, BaseResponseDto<StartProcessResponse>>
    {

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
