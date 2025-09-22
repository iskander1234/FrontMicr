using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Enum;
using BpmBaseApi.Shared.Models.Camunda;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using System.Text.Json;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    /// <summary>
    /// ITSM-версия Send: передаём одну переменную itsmLevel со значением
    /// "Экстремальное" | "Стандартное" | "Нормальное". Иначе — без переменных.
    /// Остальная логика: просто «вперёд».
    /// </summary>
    public class SendProcessITSMCommandHandler(
        IUnitOfWork unitOfWork,
        ICamundaService camundaService,
        IProcessTaskService processTaskService)
        : IRequestHandler<SendProcessCommand, BaseResponseDto<SendProcessResponse>>
    {
        public async Task<BaseResponseDto<SendProcessResponse>> Handle(
            SendProcessCommand command,
            CancellationToken ct)
        {
            // 1) Текущая подзадача должна быть Pending
            var currentTask = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                                  ct, t => t.Id == command.TaskId && t.Status == "Pending")
                              ?? throw new HandlerException("Задача не найдена или уже обработана", ErrorCodesEnum.Business);

            var processData = await unitOfWork.ProcessDataRepository.GetByIdAsync(ct, currentTask.ProcessDataId)
                             ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

            // 2) Родитель (чаще system) — будем завершать его или поднимать дальше
            var parentTask = await unitOfWork.ProcessTaskRepository
                .GetByIdAsync(ct, currentTask.ParentTaskId!.Value);

            // 3) Собираем одну переменную itsmLevel (опционально)
            var variables = BuildItsmLevelVariables(command.PayloadJson);

            // 4) Если родительская Camunda-задача на system — сабмитим её
            if (string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
            {
                var claimed = await camundaService.CamundaClaimedTasks(
                                  processData.ProcessInstanceId, parentTask.BlockCode)
                              ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);

                var submit = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
                {
                    TaskId = claimed.TaskId,
                    Variables = variables // может быть пустым словарём — это ок
                });

                if (!submit.Success)
                    throw new HandlerException($"Ошибка при отправке Msg:{submit.Msg}", ErrorCodesEnum.Camunda);

                // Закрываем parent, т.к. он был на system
                await processTaskService.FinalizeTaskAsync(parentTask, ct);
            }
            else
            {
                // Если родитель не system — просто делаем его Pending (без Camunda)
                await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
                {
                    EntityId = parentTask.Id,
                    Status = "Pending"
                }, ct);

                if (!string.IsNullOrWhiteSpace(parentTask.AssigneeCode))
                    await processTaskService.RefreshUserTaskCacheAsync(parentTask.AssigneeCode, ct);
            }

            // 5) Лог + закрытие текущей подзадачи
            await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, ct);
            await processTaskService.FinalizeTaskAsync(currentTask, ct);
            await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, ct);

            return new BaseResponseDto<SendProcessResponse>
            {
                Data = new SendProcessResponse { Success = true }
            };
        }

        /// <summary>
        /// Пытаемся вытащить уровень из payload:
        /// ключи: itsmLevel | level | priority (регистронезависимо).
        /// Разрешённые значения (регистронезависимо):
        ///   - "Экстремальное"
        ///   - "Стандартное"
        ///   - "Нормальное"
        /// Если попали — вернём { itsmLevel = "<значение>" }, иначе пусто.
        /// </summary>
        private static Dictionary<string, object> BuildItsmLevelVariables(Dictionary<string, object>? payload)
        {
            if (payload is null) return new();

            static string? readStr(Dictionary<string, object> p, string key)
            {
                if (!p.TryGetValue(key, out var v) || v is null) return null;
                var json = JsonSerializer.Serialize(v);
                return JsonSerializer.Deserialize<string>(json, new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
            }

            string? level = readStr(payload, "itsmLevel")
                            ?? readStr(payload, "level")
                            ?? readStr(payload, "priority");

            if (string.IsNullOrWhiteSpace(level)) return new();

            // Нормализуем и валидируем русские значения
            var normalized = level.Trim().ToLowerInvariant();

            // Разрешаем и без ё:
            normalized = normalized
                .Replace('ё', 'е'); // "экстремальное" vs "экстремальное"

            string? canonical = normalized switch
            {
                "экстремальное" => "Экстремальное",
                "стандартное"   => "Стандартное",
                "нормальное"    => "Нормальное",
                _               => null
            };

            return canonical is null
                ? new()
                : new Dictionary<string, object> { ["itsmLevel"] = canonical };
        }
    }
}
