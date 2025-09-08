using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
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

            // 2) Номер заявки (как было)
            var requestNumber = await helperService
                .GenerateRequestNumberAsync(command.ProcessCode, cancellationToken);

            // 3) Готовим payload
            var jsonOptions = new JsonSerializerOptions { Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping };
            var payloadJson = JsonSerializer.Serialize(command.Payload, jsonOptions);

            var regData        = payloadReader.ReadSection<RegData>(command.Payload, "regData");
            var processDataDto = payloadReader.ReadSection<ProcessDataDto>(command.Payload, "processData");
            var regNumberFromPayload = TryGetRegNumberFromPayload(payloadJson);

            // ===== A) Если есть Draft с таким RegNumber — обновить поля и стартовать его =====
            if (!string.IsNullOrWhiteSpace(regNumberFromPayload))
            {
                var draft = await unitOfWork.ProcessDataRepository.GetByFilterAsync(
                    cancellationToken,
                    p => p.ProcessCode == process.ProcessCode
                         && p.RegNumber == regNumberFromPayload
                         && p.StatusCode == "Draft"
                );

                if (draft is not null)
                {
                    // Обновляем ТОЛЬКО нужные поля событием (RegNumber не трогаем)
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

                    // Camunda + связывающее событие
                    await StartCamundaAndLinkAsync(draft.Id, cancellationToken);

                    // Файлы + История
                    await AddFilesAsync(draft.Id, payloadJson, cancellationToken);
                    await WriteHistoryStartAsync(draft.Id, draft.RegNumber, payloadJson, cancellationToken);

                    return new BaseResponseDto<StartProcessResponse>
                    {
                        Data = new StartProcessResponse { ProcessGuid = draft.Id, RegNumber = draft.RegNumber }
                    };
                }
            }

            // ===== B) Черновика нет — как было: создаём Started и стартуем
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
            await AddFilesAsync(createdEvt.EntityId, payloadJson, cancellationToken);
            await WriteHistoryStartAsync(createdEvt.EntityId, requestNumber, payloadJson, cancellationToken);

            return new BaseResponseDto<StartProcessResponse>
            {
                Data = new StartProcessResponse { ProcessGuid = createdEvt.EntityId, RegNumber = requestNumber }
            };

            // ---------- локальные функции, чтобы не дублировать код ----------

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

            async Task AddFilesAsync(Guid processDataId, string payload, CancellationToken ct)
            {
                try
                {
                    var root  = JsonNode.Parse(payload)!.AsObject();
                    var files = root["files"] as JsonArray;
                    if (files is null) return;

                    foreach (var node in files.OfType<JsonObject>())
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
                            ProcessDataId = processDataId,
                            FileId        = fileId,
                            FileName      = fileName,
                            FileType      = fileType
                        }, ct);
                    }
                }
                catch
                {
                    // опционально: логирование
                }
            }

            async Task WriteHistoryStartAsync(Guid processDataId, string regNumber, string payload, CancellationToken ct)
            {
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
}
