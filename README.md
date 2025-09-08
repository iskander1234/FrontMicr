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
    // Никакого ICamundaService тут нет
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
                ?? throw new HandlerException($"Процесс с кодом {command.ProcessCode} не найден", ErrorCodesEnum.Business);

            var options    = new JsonSerializerOptions { Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping };
            var payloadJson = JsonSerializer.Serialize(command.Payload, options);

            var regData        = payloadReader.ReadSection<RegData>(command.Payload, "regData");
            var processDataDto = payloadReader.ReadSection<ProcessDataDto>(command.Payload, "processData");

            // 1) Пытаемся взять номер из payload.regData.regnum
            var regnumFromPayload = TryGetRegnumFromPayload(payloadJson);

            if (!string.IsNullOrWhiteSpace(regnumFromPayload))
            {
                // A) Если есть Draft с этим номером — ОБНОВЛЯЕМ (без Camunda)
                var existing = await unitOfWork.ProcessDataRepository.GetByFilterAsync(
                    cancellationToken,
                    p => p.ProcessCode == process.ProcessCode
                         && p.RegNumber == regnumFromPayload
                );

                if (existing is not null)
                {
                    if (!string.Equals(existing.StatusCode, "Draft", StringComparison.OrdinalIgnoreCase))
                        throw new HandlerException("Заявка уже не в Черновике, повторное сохранение недоступно.",
                            ErrorCodesEnum.Business);

                    // Синхронизация номера внутри payload (на всякий случай)
                    payloadJson = SetRegnumInPayload(payloadJson, existing.RegNumber, options);

                    await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataEditedEvent
                    {
                        EntityId      = existing.Id,
                        StatusCode    = "Draft",
                        StatusName    = "Черновик",
                        PayloadJson   = payloadJson,
                        InitiatorCode = regData?.UserCode,
                        InitiatorName = regData?.UserName,
                        Title         = processDataDto?.DocumentTitle ?? existing.Title
                    }, cancellationToken);

                    return new BaseResponseDto<StartProcessResponse>
                    {
                        Data = new StartProcessResponse
                        {
                            ProcessGuid = existing.Id,
                            RegNumber   = existing.RegNumber
                        }
                    };
                }

                // если номер в payload есть, но записи нет — создадим НОВЫЙ черновик со СВОИМ номером
                // (чтобы исключить коллизии и «самодельные» номера от клиента)
            }

            // B) Генерим номер и создаём новый Draft
            var requestNumber = await helperService.GenerateRequestNumberAsync(command.ProcessCode, cancellationToken);
            payloadJson = SetRegnumInPayload(payloadJson, requestNumber, options);

            var created = new ProcessDataCreatedEvent
            {
                ProcessId     = process.Id,
                ProcessCode   = process.ProcessCode,
                ProcessName   = process.ProcessName,
                RegNumber     = requestNumber,
                InitiatorCode = regData.UserCode,
                InitiatorName = regData.UserName,
                StatusCode    = "Draft",
                StatusName    = "Черновик",
                PayloadJson   = payloadJson,
                Title         = processDataDto.DocumentTitle
            };

            await unitOfWork.ProcessDataRepository.RaiseEvent(created, cancellationToken);

            return new BaseResponseDto<StartProcessResponse>
            {
                Data = new StartProcessResponse
                {
                    ProcessGuid = created.EntityId,
                    RegNumber   = requestNumber
                }
            };
        }

        // ===== helpers =====

        private static string? TryGetRegnumFromPayload(string payload)
        {
            try
            {
                var root = JsonNode.Parse(payload)!.AsObject();
                return root["regData"]?["regnum"]?.GetValue<string>()?.Trim();
            }
            catch { return null; }
        }

        private static string SetRegnumInPayload(string payload, string regnum, JsonSerializerOptions opts)
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
