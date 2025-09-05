using MediatR;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;
using System.Collections.Generic;

namespace BpmBaseApi.Shared.Commands.Process
{
    public record SaveProcessCommand(
        Guid ProcessGuid,                // обязательный! сохраняем ТОЛЬКО уже созданную заявку
        string ProcessCode,              // для валидации, что код процесса совпадает
        Dictionary<string, object> Payload
    ) : IRequest<BaseResponseDto<SaveProcessResponse>>;

    public class SaveProcessResponse
    {
        public Guid ProcessGuid { get; set; }
        public string? RegNumber { get; set; }
        public string Status { get; set; } = "Draft";
    }
}



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
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using static BpmBaseApi.Shared.Models.Process.CommonData;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    public class SaveProcessCommandHandler(
        IUnitOfWork unitOfWork,
        IPayloadReaderService payloadReader
    ) : IRequestHandler<SaveProcessCommand, BaseResponseDto<SaveProcessResponse>>
    {
        public async Task<BaseResponseDto<SaveProcessResponse>> Handle(
            SaveProcessCommand command,
            CancellationToken cancellationToken)
        {
            // 0) Проверяем, что заявка уже существует — если нет, в черновик отправлять нельзя
            var processData = await unitOfWork.ProcessDataRepository
                .GetByFilterAsync(cancellationToken, p => p.Id == command.ProcessGuid)
                ?? throw new HandlerException("Заявка не найдена — нельзя сохранить в черновик", ErrorCodesEnum.Business);

            // (опционально) защитимся от несоответствия кода процесса
            if (!string.Equals(processData.ProcessCode, command.ProcessCode, StringComparison.OrdinalIgnoreCase))
                throw new HandlerException("Код процесса не совпадает с заявкой", ErrorCodesEnum.Business);

            // 1) Готовим payload
            var options = new JsonSerializerOptions { Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping };
            var payloadJson = JsonSerializer.Serialize(command.Payload, options);

            var regData = payloadReader.ReadSection<RegData>(command.Payload, "regData");
            var processDataDto = payloadReader.ReadSection<BpmBaseApi.Shared.Models.Process.ProcessDataDto>(command.Payload, "processData");

            // 2) Обновляем payload/заголовок заявки (через событие)
            await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataPayloadChangedEvent
            {
                EntityId   = processData.Id,
                PayloadJson= payloadJson,
                Title      = processDataDto?.DocumentTitle ?? processData.Title
            }, cancellationToken);

            // 3) Меняем статус на Draft (через событие)
            await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataStatusChangedEvent
            {
                EntityId   = processData.Id,
                StatusCode = "Draft",
                StatusName = "Черновик"
            }, cancellationToken);

            // 4) Сохраняем файлы (как в Start). Никакой Camunda.
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
                        throw new HandlerException("Каждый файл должен иметь fileName и fileType", ErrorCodesEnum.Business);

                    if (string.IsNullOrWhiteSpace(fileIdStr) || !Guid.TryParse(fileIdStr, out var fileId))
                        throw new HandlerException("fileId должен быть корректным GUID", ErrorCodesEnum.Business);

                    // Здесь простая «добавка» файлов, как в Start.
                    // Если нужно "заменить" набор — предварительно почистите записи файлов по ProcessDataId.
                    await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileCreatedEvent
                    {
                        EntityId     = Guid.NewGuid(),
                        ProcessDataId= processData.Id,
                        FileId       = fileId,
                        FileName     = fileName,
                        FileType     = fileType
                    }, cancellationToken);
                }
            }

            // 5) Пишем историю (для аудита «сохранён как черновик»)
            await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(new ProcessTaskHistoryCreatedEvent
            {
                ProcessDataId = processData.Id,
                TaskId        = processData.Id,                 // без Camunda
                Action        = "Save",                         // или "SaveDraft" — как удобнее
                BlockName     = "Регистрационная форма",
                Timestamp     = DateTime.Now,
                PayloadJson   = payloadJson,
                Comment       = "",
                Description   = "Сохранено как черновик",
                ProcessCode   = processData.ProcessCode,
                ProcessName   = processData.ProcessName,
                RegNumber     = processData.RegNumber,
                InitiatorCode = regData?.UserCode ?? processData.InitiatorCode,
                InitiatorName = regData?.UserName ?? processData.InitiatorName,
                Title         = processDataDto?.DocumentTitle ?? processData.Title
            }, cancellationToken);

            return new BaseResponseDto<SaveProcessResponse>
            {
                Data = new SaveProcessResponse
                {
                    ProcessGuid = processData.Id,
                    RegNumber   = processData.RegNumber,
                    Status      = "Draft"
                }
            };
        }
    }
}
