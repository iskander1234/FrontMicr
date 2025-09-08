using System.Text.Encodings.Web;
using System.Text.Json;
using System.Text.Json.Nodes;
using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Enum;
using BpmBaseApi.Shared.Models.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using static BpmBaseApi.Shared.Models.Process.CommonData;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    // Camunda не трогаем
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

            var options    = new JsonSerializerOptions { Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping };
            var payloadJson = JsonSerializer.Serialize(command.Payload, options);

            var regData        = payloadReader.ReadSection<RegData>(command.Payload, "regData");
            var processDataDto = payloadReader.ReadSection<ProcessDataDto>(command.Payload, "processData");

            // 0) пробуем вытащить номер из входного payload
            var regnumFromPayload = TryGetRegnumFromPayload(payloadJson);

            if (!string.IsNullOrWhiteSpace(regnumFromPayload))
            {
                // A) если такой черновик существует — обновляем его
                var existingDraft = await unitOfWork.ProcessDataRepository.GetByFilterAsync(
                    cancellationToken,
                    p => p.ProcessCode == process.ProcessCode
                         && p.RegNumber == regnumFromPayload
                         && p.StatusCode == "Draft");

                if (existingDraft is not null)
                {
                    // подстрахуемся и синхронизируем номер в payload
                    payloadJson = SetRegnumInPayload(payloadJson, existingDraft.RegNumber, options);

                    await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataEditedEvent
                    {
                        EntityId      = existingDraft.Id,
                        StatusCode    = "Draft",
                        StatusName    = "Черновик",
                        PayloadJson   = payloadJson,
                        InitiatorCode = regData?.UserCode,
                        InitiatorName = regData?.UserName,
                        Title         = processDataDto?.DocumentTitle ?? existingDraft.Title
                    }, cancellationToken);

                    return new BaseResponseDto<StartProcessResponse>
                    {
                        Data = new StartProcessResponse
                        {
                            ProcessGuid = existingDraft.Id,
                            RegNumber   = existingDraft.RegNumber
                        }
                    };
                }

                // если номер есть, но запись уже не Draft — сохранять нельзя
                var notDraft = await unitOfWork.ProcessDataRepository.GetByFilterAsync(
                    cancellationToken,
                    p => p.ProcessCode == process.ProcessCode
                         && p.RegNumber == regnumFromPayload
                         && p.StatusCode != "Draft");

                if (notDraft is not null)
                    throw new HandlerException("Заявка уже не в черновике, сохранить как Draft нельзя.", ErrorCodesEnum.Business);

                // иначе — продолжим как «создать новый черновик ниже»
            }

            // B) номера нет (или по нему ничего не нашли) — создаём новый черновик, как раньше
            var newRegNumber = await helperService.GenerateRequestNumberAsync(command.ProcessCode, cancellationToken);

            // вписываем номер в payload.regData.regnum
            payloadJson = SetRegnumInPayload(payloadJson, newRegNumber, options);

            var processDataCreatedEvent = new ProcessDataCreatedEvent
            {
                ProcessId     = process.Id,
                ProcessCode   = process.ProcessCode,
                ProcessName   = process.ProcessName,
                RegNumber     = newRegNumber,
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
                    RegNumber   = newRegNumber
                }
            };

            // --- helpers ---
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
        }
    }
}
