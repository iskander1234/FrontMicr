using System.Text.Json;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Models.Camunda;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using BpmBaseApi.Domain.Entities.Event.Process;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    /// <summary>
    /// ITSM Send:
    /// - читает classification из payloadJson (processData.analyst.classificationCode | classification | level | priority)
    /// - всегда устанавливает переменную на инстансе процесса
    /// - далее как в обычном Send: если родитель system — submit, иначе поднимаем его в Pending
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

            var parentTask = await unitOfWork.ProcessTaskRepository
                .GetByIdAsync(ct, currentTask.ParentTaskId!.Value);

            // 2) Подготовим переменные Camunda
            var vars = BuildClassificationVariables(command.PayloadJson);

            // 2.1 Всегда сохраняем их на инстансе (даже если родитель не system),
            // чтобы выражения в шлюзах не падали из-за отсутствующей переменной
            if (vars.Count > 0)
            {
                await camundaService.CamundaSetProcessVariables(processData.ProcessInstanceId, vars);
            }

            // 3) Если родительская Camunda-задача на system — submit
            if (string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
            {
                var claimed = await camundaService.CamundaClaimedTasks(
                                  processData.ProcessInstanceId, parentTask.BlockCode)
                              ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);

                var submit = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
                {
                    TaskId = claimed.TaskId,
                    // Можно передать пустой словарь — мы уже проставили переменные выше.
                    // Но передать повторно безопасно:
                    Variables = vars
                });

                if (!submit.Success)
                    throw new HandlerException($"Ошибка при отправке Msg:{submit.Msg}", ErrorCodesEnum.Camunda);

                await processTaskService.FinalizeTaskAsync(parentTask, ct);
            }
            else
            {
                // Родитель не system — просто поднимем его в Pending
                await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
                {
                    EntityId = parentTask.Id,
                    Status = "Pending"
                }, ct);

                if (!string.IsNullOrWhiteSpace(parentTask.AssigneeCode))
                    await processTaskService.RefreshUserTaskCacheAsync(parentTask.AssigneeCode, ct);
            }

            // 4) Лог + закрытие текущей подзадачи
            await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, ct);
            await processTaskService.FinalizeTaskAsync(currentTask, ct);
            await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, ct);

            return new BaseResponseDto<SendProcessResponse>
            {
                Data = new SendProcessResponse { Success = true }
            };
        }

        /// <summary>
        /// Собираем переменную classification (Emergency|Standard|Normal) из payloadJson.
        /// Приоритет: processData.analyst.classificationCode -> classification -> level -> priority
        /// </summary>
        private static Dictionary<string, object> BuildClassificationVariables(Dictionary<string, object>? payload)
        {
            if (payload is null) return new();

            static string? ReadStr(Dictionary<string, object> p, string key)
            {
                if (!p.TryGetValue(key, out var v) || v is null) return null;
                var json = JsonSerializer.Serialize(v);
                return JsonSerializer.Deserialize<string>(json, new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
            }

            // Достаём processData.analyst.classificationCode
            string? fromAnalyst = null;
            if (payload.TryGetValue("processData", out var pdRaw) && pdRaw is not null)
            {
                var pdJson = JsonSerializer.Serialize(pdRaw);
                var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(pdJson,
                    new JsonSerializerOptions { PropertyNameCaseInsensitive = true });

                if (pd != null && pd.TryGetValue("analyst", out var aRaw) && aRaw is not null)
                {
                    var aJson = JsonSerializer.Serialize(aRaw);
                    var analyst = JsonSerializer.Deserialize<Dictionary<string, object>>(aJson,
                        new JsonSerializerOptions { PropertyNameCaseInsensitive = true });

                    if (analyst != null)
                        fromAnalyst = ReadStr(analyst, "classificationCode");
                }
            }

            var level = fromAnalyst
                        ?? ReadStr(payload, "classification")
                        ?? ReadStr(payload, "itsmLevel")
                        ?? ReadStr(payload, "level")
                        ?? ReadStr(payload, "priority");

            if (string.IsNullOrWhiteSpace(level)) return new();

            var canonical = level.Trim().ToLowerInvariant() switch
            {
                "emergency" => "Emergency",
                "standard"  => "Standard",
                "normal"    => "Normal",
                _ => null
            };

            return canonical is null ? new() : new() { ["classification"] = canonical };
        }
    }
}
