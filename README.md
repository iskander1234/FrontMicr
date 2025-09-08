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
                          ?? throw new HandlerException(
                              $"Процесс с кодом {command.ProcessCode} не найден",
                              ErrorCodesEnum.Business);

            var requestNumber = await helperService
                .GenerateRequestNumberAsync(command.ProcessCode, cancellationToken);

            var options = new JsonSerializerOptions { Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping };
            var payloadJson = JsonSerializer.Serialize(command.Payload, options);

            var regData = payloadReader.ReadSection<RegData>(command.Payload, "regData");
            var processDataDto = payloadReader.ReadSection<ProcessDataDto>(command.Payload, "processData");

            // NEW: вписываем номер в payload.regData.regnum
            payloadJson = SetRegnumInPayload(payloadJson, requestNumber, options);

            var processDataCreatedEvent = new ProcessDataCreatedEvent
            {
                ProcessId = process.Id,
                ProcessCode = process.ProcessCode,
                ProcessName = process.ProcessName,
                RegNumber = requestNumber,
                InitiatorCode = regData.UserCode,
                InitiatorName = regData.UserName,
                StatusCode = "Draft",
                StatusName = "Черновик",
                PayloadJson = payloadJson,
                Title = processDataDto.DocumentTitle
            };

            await unitOfWork.ProcessDataRepository
                .RaiseEvent(processDataCreatedEvent, cancellationToken);

            return new BaseResponseDto<StartProcessResponse>
            {
                Data = new StartProcessResponse
                {
                    ProcessGuid = processDataCreatedEvent.EntityId,
                    RegNumber = requestNumber
                }
            };

            // helper (NEW)
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
