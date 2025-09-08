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
using System.Text.Json.Nodes;
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
        public async Task<BaseResponseDto<StartProcessResponse>> Handle(
            StartProcessCommand command,
            CancellationToken cancellationToken)
        {
           
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

            var regData = payloadReader.ReadSection<RegData>(command.Payload, "regData");
            var processDataDto = payloadReader.ReadSection<ProcessDataDto>(command.Payload, "processData");

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
                Title = processDataDto.DocumentTitle
            };

            await unitOfWork.ProcessDataRepository
                .RaiseEvent(processDataCreatedEvent, cancellationToken);

            var camundaVariables = new Dictionary<string, object>
            {
                { "processGuid", processDataCreatedEvent.EntityId.ToString() }
            };

            var response = await camundaService.CamundaStartProcess(
                new CamundaStartProcessRequest { processCode = command.ProcessCode, variables = camundaVariables });

            await unitOfWork
                .ProcessDataRepository
                .RaiseEvent
                (
                    new ProcessDataProcessInstanseIdChangedEvent()
                    {
                        EntityId = processDataCreatedEvent.EntityId,
                        ProcessInstanceId = response
                    },
                    cancellationToken
                );

            // --- 2) NEW: Парсим из готовой строки payloadJson массив files вручную ---
            var root = JsonNode.Parse(payloadJson)!.AsObject();
            var filesArray = root["files"] as JsonArray;

            if (filesArray != null)
            {
                foreach (var node in filesArray.OfType<JsonObject>())
                {
                    // извлекаем fileName и fileType
                    var fileIdStr = node["fileId"]?.GetValue<string>();   // <-- NEW
                    var fileName = node["fileName"]?.GetValue<string>();
                    var fileType = node["fileType"]?.GetValue<string>();

                    if (string.IsNullOrWhiteSpace(fileName) || string.IsNullOrWhiteSpace(fileType))
                    {
                        throw new HandlerException(
                            "Каждый объект в \"files\" должен иметь непустые fileName и fileType",
                            ErrorCodesEnum.Business);
                    }
                    // Жёсткая проверка, что fileId — корректный GUID
                    if (string.IsNullOrWhiteSpace(fileIdStr) || !Guid.TryParse(fileIdStr, out var fileId))
                        throw new HandlerException("Каждый объект в \"files\" должен иметь корректный GUID в поле fileId", ErrorCodesEnum.Business);

                    var fileEvent = new ProcessFileCreatedEvent
                    {
                        EntityId = Guid.NewGuid(),                                // PK → Id записи
                        ProcessDataId = processDataCreatedEvent.EntityId,              // FK на ProcessData
                        FileId   = fileId,                               // <-- NEW
                        FileName = fileName,
                        FileType = fileType
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
                ProcessDataId = processDataCreatedEvent.EntityId,
                TaskId = processDataCreatedEvent.EntityId,
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
            await unitOfWork.ProcessTaskHistoryRepository
                .RaiseEvent(processTaskHistoryCreatedEvent, cancellationToken);

            return new BaseResponseDto<StartProcessResponse>
            {
                Data = new StartProcessResponse
                {
                    ProcessGuid = processDataCreatedEvent.EntityId,
                    RegNumber = requestNumber
                }
            };
        }
}
}
