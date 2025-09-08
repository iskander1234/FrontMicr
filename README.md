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
            // 1) Процесс
            var process = await unitOfWork.ProcessRepository
                .GetByFilterAsync(cancellationToken, p => p.ProcessCode == command.ProcessCode)
                ?? throw new HandlerException(
                    $"Процесс с кодом {command.ProcessCode} не найден",
                    ErrorCodesEnum.Business);

            // 2) Генерация номера — оставляю как было (не ломаем твою струю)
            var requestNumber = await helperService
                .GenerateRequestNumberAsync(command.ProcessCode, cancellationToken);

            var options = new JsonSerializerOptions
            {
                Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping
            };

            // 3) Payload → строка
            var payloadJson = JsonSerializer.Serialize(command.Payload, options);

            var regData        = payloadReader.ReadSection<RegData>(command.Payload, "regData");
            var processDataDto = payloadReader.ReadSection<ProcessDataDto>(command.Payload, "processData");

            // 3.1) попытаемся взять regNumber из payload
            string? regNumberFromPayload = null;
            try
            {
                var rootTry = JsonNode.Parse(payloadJson)!.AsObject();
                regNumberFromPayload = rootTry["regData"]?["regNumber"]?.GetValue<string>();
            }
            catch { /* ок */ }

            // ===== A) Если есть Draft с таким RegNumber — ОБНОВИТЬ и СТАРТОВАТЬ ЕГО =====
            if (!string.IsNullOrWhiteSpace(regNumberFromPayload))
            {
                var draft = await unitOfWork.ProcessDataRepository.GetByFilterAsync(
                    cancellationToken,
                    p => p.ProcessCode == process.ProcessCode
                         && p.RegNumber == regNumberFromPayload
                         && p.StatusCode == "Draft"
                    // по желанию добавь && p.InitiatorCode == regData.UserCode
                );

                if (draft is not null)
                {
                    // --- обновляем только требуемые поля (RegNumber не трогаем)
                    draft.StatusCode    = "Started";
                    draft.StatusName    = "В работе";
                    draft.PayloadJson   = payloadJson;
                    draft.InitiatorCode = regData?.UserCode ?? draft.InitiatorCode;
                    draft.InitiatorName = regData?.UserName ?? draft.InitiatorName;
                    draft.Title         = processDataDto?.DocumentTitle ?? draft.Title;

                    // если у репозитория другой метод апдейта — подставь свой
                    await unitOfWork.ProcessDataRepository.UpdateAsync(draft, cancellationToken);

                    // --- стартуем Camunda на этом же processGuid
                    var pi = await camundaService.CamundaStartProcess(
                        new CamundaStartProcessRequest
                        {
                            processCode = command.ProcessCode,
                            variables   = new Dictionary<string, object> { { "processGuid", draft.Id.ToString() } }
                        });

                    await unitOfWork.ProcessDataRepository.RaiseEvent(
                        new ProcessDataProcessInstanseIdChangedEvent
                        {
                            EntityId          = draft.Id,
                            ProcessInstanceId = pi
                        },
                        cancellationToken
                    );

                    // --- файлы из payload (как у тебя): добавляем записи
                    try
                    {
                        var rootDraft  = JsonNode.Parse(payloadJson)!.AsObject();
                        var filesDraft = rootDraft["files"] as JsonArray;
                        if (filesDraft != null)
                        {
                            foreach (var node in filesDraft.OfType<JsonObject>())
                            {
                                var fileIdStr = node["fileId"]?.GetValue<string>();
                                var fileName  = node["fileName"]?.GetValue<string>();
                                var fileType  = node["fileType"]?.GetValue<string>();

                                if (string.IsNullOrWhiteSpace(fileName) || string.IsNullOrWhiteSpace(fileType))
                                    throw new HandlerException("Каждый объект в \"files\" должен иметь непустые fileName и fileType", ErrorCodesEnum.Business);
                                if (string.IsNullOrWhiteSpace(fileIdStr) || !Guid.TryParse(fileIdStr, out var fileId))
                                    throw new HandlerException("Каждый объект в \"files\" должен иметь корректный GUID в поле fileId", ErrorCodesEnum.Business);

                                await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileCreatedEvent
                                {
                                    EntityId      = Guid.NewGuid(),
                                    ProcessDataId = draft.Id,
                                    FileId        = fileId,
                                    FileName      = fileName,
                                    FileType      = fileType
                                }, cancellationToken);
                            }
                        }
                    }
                    catch { /* опционально логировать */ }

                    // --- история
                    await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(new ProcessTaskHistoryCreatedEvent
                    {
                        ProcessDataId = draft.Id,
                        TaskId        = draft.Id,
                        Action        = ProcessAction.Start.ToString(),
                        BlockName     = "Регистрационная форма",
                        Timestamp     = DateTime.Now,
                        PayloadJson   = payloadJson,
                        Comment       = "",
                        Description   = "",
                        ProcessCode   = command.ProcessCode,
                        ProcessName   = process.ProcessName,
                        RegNumber     = draft.RegNumber,
                        InitiatorCode = regData.UserCode,
                        InitiatorName = regData.UserName,
                        Title         = processDataDto.DocumentTitle
                    }, cancellationToken);

                    return new BaseResponseDto<StartProcessResponse>
                    {
                        Data = new StartProcessResponse
                        {
                            ProcessGuid = draft.Id,
                            RegNumber   = draft.RegNumber
                        }
                    };
                }
            }

            // ===== B) Черновика нет — текущее поведение (как было)
            var processDataCreatedEvent = new ProcessDataCreatedEvent
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

            // --- файлы (как было)
            var root = JsonNode.Parse(payloadJson)!.AsObject();
            var filesArray = root["files"] as JsonArray;

            if (filesArray != null)
            {
                foreach (var node in filesArray.OfType<JsonObject>())
                {
                    var fileIdStr = node["fileId"]?.GetValue<string>();
                    var fileName  = node["fileName"]?.GetValue<string>();
                    var fileType  = node["fileType"]?.GetValue<string>();

                    if (string.IsNullOrWhiteSpace(fileName) || string.IsNullOrWhiteSpace(fileType))
                        throw new HandlerException(
                            "Каждый объект в \"files\" должен иметь непустые fileName и fileType",
                            ErrorCodesEnum.Business);
                    if (string.IsNullOrWhiteSpace(fileIdStr) || !Guid.TryParse(fileIdStr, out var fileId))
                        throw new HandlerException("Каждый объект в \"files\" должен иметь корректный GUID в поле fileId", ErrorCodesEnum.Business);

                    var fileEvent = new ProcessFileCreatedEvent
                    {
                        EntityId      = Guid.NewGuid(),
                        ProcessDataId = processDataCreatedEvent.EntityId,
                        FileId        = fileId,
                        FileName      = fileName,
                        FileType      = fileType
                    };
                    await unitOfWork.ProcessFileRepository
                        .RaiseEvent(fileEvent, cancellationToken);
                }
            }

            // --- история (как было)
            var processTaskHistoryCreatedEvent = new ProcessTaskHistoryCreatedEvent
            {
                ProcessDataId = processDataCreatedEvent.EntityId,
                TaskId        = processDataCreatedEvent.EntityId,
                Action        = ProcessAction.Start.ToString(),
                BlockName     = "Регистрационная форма",
                Timestamp     = DateTime.Now,
                PayloadJson   = payloadJson,
                Comment       = "",
                Description   = "",
                ProcessCode   = command.ProcessCode,
                ProcessName   = process.ProcessName,
                RegNumber     = requestNumber,
                InitiatorCode = regData.UserCode,
                InitiatorName = regData.UserName,
                Title         = processDataDto.DocumentTitle
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
}
