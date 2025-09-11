using System.Text.Json;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    public class NeedsTranslationCommandHandler
        : IRequestHandler<NeedsTranslationCommand, BaseResponseDto<NeedsTranslationResponse>>
    {
        private readonly IUnitOfWork _unitOfWork;

        public NeedsTranslationCommandHandler(IUnitOfWork unitOfWork)
        {
            _unitOfWork = unitOfWork;
        }

        public async Task<BaseResponseDto<NeedsTranslationResponse>> Handle(
            NeedsTranslationCommand command, CancellationToken ct)
        {
            if (command.ProcessGuid == Guid.Empty)
            {
                return new BaseResponseDto<NeedsTranslationResponse>
                {
                    ErrorCode = 400,
                    Message = "ProcessGuid не задан.",
                    Data = new NeedsTranslationResponse { Result = false }
                };
            }

            // Имя репозитория — под вашу фактическую реализацию:
            // чаще всего это ProcessDataRepository
            var entity = await _unitOfWork.ProcessDataRepository
                .GetByFilterAsync(ct, x => x.Id == command.ProcessGuid);

            if (entity == null)
            {
                return new BaseResponseDto<NeedsTranslationResponse>
                {
                    ErrorCode = 404,
                    Message = $"ProcessData {command.ProcessGuid} не найден.",
                    Data = new NeedsTranslationResponse { Result = false }
                };
            }

            if (string.IsNullOrWhiteSpace(entity.PayloadJson))
            {
                // Нет JSON — трактуем как нет перевода (false)
                return new BaseResponseDto<NeedsTranslationResponse>
                {
                    Data = new NeedsTranslationResponse { Result = false }
                };
            }

            try
            {
                using var doc = JsonDocument.Parse(entity.PayloadJson);
                var root = doc.RootElement;

                if (TryGetPropertyCaseInsensitive(root, "processData", out var processData) &&
                    TryGetPropertyCaseInsensitive(processData, "translatable", out var translatable))
                {
                    var result = translatable.ValueKind == JsonValueKind.True
                                ? true
                                : translatable.ValueKind == JsonValueKind.False
                                  ? false
                                  : false; // любое не-bool значение трактуем как false

                    return new BaseResponseDto<NeedsTranslationResponse>
                    {
                        Data = new NeedsTranslationResponse { Result = result }
                    };
                }

                // Нет поля processData.translatable — трактуем как false
                return new BaseResponseDto<NeedsTranslationResponse>
                {
                    Data = new NeedsTranslationResponse { Result = false }
                };
            }
            catch (JsonException)
            {
                // Некорректный JSON — вернём конфликт с описанием
                return new BaseResponseDto<NeedsTranslationResponse>
                {
                    ErrorCode = 409,
                    Message = "PayloadJson содержит некорректный JSON.",
                    Data = new NeedsTranslationResponse { Result = false }
                };
            }
        }

        private static bool TryGetPropertyCaseInsensitive(JsonElement element, string name, out JsonElement value)
        {
            if (element.ValueKind == JsonValueKind.Object)
            {
                foreach (var p in element.EnumerateObject())
                {
                    if (string.Equals(p.Name, name, StringComparison.OrdinalIgnoreCase))
                    {
                        value = p.Value;
                        return true;
                    }
                }
            }
            value = default;
            return false;
        }
    }
}
