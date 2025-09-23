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
using BpmBaseApi.Shared.Enum;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    /// <summary>
    /// ITSM Send:
    /// - читает classification из payloadJson (processData.analyst.classificationCode | classification | itsmLevel | level | priority)
    /// - при системном parent: отправляет в Camunda submit с переменной classification
    /// - для Level1 дополнительно шлёт булевую переменную "Line 1" по execution (executed/toSecondLine)
    /// - иначе поднимает родителя в Pending
    /// </summary>
    public class SendProcessITSMCommandHandler(
        IUnitOfWork unitOfWork,
        ICamundaService camundaService,
        IProcessTaskService processTaskService)
        : IRequestHandler<SendItsmProcessCommand, BaseResponseDto<SendProcessResponse>>
    {
        public async Task<BaseResponseDto<SendProcessResponse>> Handle(
            SendItsmProcessCommand command, CancellationToken ct)
        {
            // 1) текущая подзадача должна быть Pending
            var currentTask = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                                  ct, t => t.Id == command.TaskId && t.Status == "Pending")
                              ?? throw new HandlerException("Задача не найдена или уже обработана", ErrorCodesEnum.Business);

            var processData = await unitOfWork.ProcessDataRepository.GetByIdAsync(ct, currentTask.ProcessDataId)
                              ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

            var parentTask = await unitOfWork.ProcessTaskRepository.GetByIdAsync(ct, currentTask.ParentTaskId!.Value);

            // 2) читаем и нормализуем classification (оставляем старую логику)
            var classification = ReadClassification(command.PayloadJson);
            if (string.IsNullOrWhiteSpace(classification))
                throw new HandlerException("Не удалось определить classification (ожидается Emergency|Standard|Normal).", ErrorCodesEnum.Business);

            if (string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
            {
                var claimed = await camundaService.CamundaClaimedTasks(
                                  processData.ProcessInstanceId, parentTask.BlockCode)
                              ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);

                var variables = new Dictionary<string, object>
                {
                    ["classification"] = classification
                };

                // ---- ДОБАВКА: ветка второй схемы для Level1 ----
                if (Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var stage) && stage == ProcessStage.Level1)
                {
                    // читаем processData.level1.formData.execution ("executed"|"toSecondLine")
                    var execution = ReadLevel1Execution(command.PayloadJson);
                    bool line1 = execution switch
                    {
                        "executed"     => true,
                        "toSecondLine" => false,
                        _              => true // дефолт (при желании поменяй на false)
                    };

                    // имя переменной должно совпадать с BPMN (у тебя на схеме "Line 1")
                    variables["Line 1"] = line1;
                }
                // ---- КОНЕЦ ДОБАВКИ ----

                var submit = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
                {
                    TaskId    = claimed.TaskId,
                    Variables = variables
                });

                if (!submit.Success)
                    throw new HandlerException($"Ошибка при отправке Msg:{submit.Msg}", ErrorCodesEnum.Camunda);

                // сначала лог/закрыть текущую
                await processTaskService.LogHistoryItsmAsync(currentTask, command, currentTask.AssigneeCode, ct);
                await processTaskService.FinalizeTaskAsync(currentTask, ct);
                await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, ct);

                // затем закрыть родителя (system)
                await processTaskService.FinalizeTaskAsync(parentTask, ct);
            }
            else
            {
                await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
                {
                    EntityId = parentTask.Id,
                    Status   = "Pending"
                }, ct);

                if (!string.IsNullOrWhiteSpace(parentTask.AssigneeCode))
                    await processTaskService.RefreshUserTaskCacheAsync(parentTask.AssigneeCode, ct);

                await processTaskService.LogHistoryItsmAsync(currentTask, command, currentTask.AssigneeCode, ct);
                await processTaskService.FinalizeTaskAsync(currentTask, ct);
                await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, ct);
            }

            // защита от двойного лога: если выше уже логировали — второй вызов можешь удалить
            await processTaskService.LogHistoryItsmAsync(currentTask, command, currentTask.AssigneeCode, ct);
            await processTaskService.FinalizeTaskAsync(currentTask, ct);
            await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, ct);

            return new BaseResponseDto<SendProcessResponse>
            {
                Data = new SendProcessResponse { Success = true }
            };
        }

        /// <summary>
        /// Старое правило: вернуть "Emergency"|"Standard"|"Normal"
        /// </summary>
        private static string? ReadClassification(Dictionary<string, object>? payload)
        {
            if (payload is null) return null;

            static string? ReadStr(Dictionary<string, object> p, string key)
            {
                if (!p.TryGetValue(key, out var v) || v is null) return null;
                var json = JsonSerializer.Serialize(v);
                return JsonSerializer.Deserialize<string>(json, new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
            }

            string? fromAnalyst = null;
            if (payload.TryGetValue("processData", out var pdRaw) && pdRaw is not null)
            {
                var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(
                    JsonSerializer.Serialize(pdRaw), new JsonSerializerOptions { PropertyNameCaseInsensitive = true });

                if (pd != null && pd.TryGetValue("analyst", out var aRaw) && aRaw is not null)
                {
                    var analyst = JsonSerializer.Deserialize<Dictionary<string, object>>(
                        JsonSerializer.Serialize(aRaw), new JsonSerializerOptions { PropertyNameCaseInsensitive = true });

                    if (analyst != null)
                        fromAnalyst = ReadStr(analyst, "classificationCode");
                }
            }

            var raw = fromAnalyst
                      ?? ReadStr(payload, "classification")
                      ?? ReadStr(payload, "itsmLevel")
                      ?? ReadStr(payload, "level")
                      ?? ReadStr(payload, "priority");

            if (string.IsNullOrWhiteSpace(raw)) return null;

            var n = raw.Trim().ToLowerInvariant();
            return n switch
            {
                "emergency" => "Emergency",
                "standard"  => "Standard",
                "normal"    => "Normal",
                _           => null
            };
        }

        /// <summary>
        /// Для второй схемы (Level1): processData.level1.formData.execution
        /// </summary>
        private static string? ReadLevel1Execution(Dictionary<string, object>? payload)
        {
            try
            {
                if (payload is null) return null;

                if (!payload.TryGetValue("payload", out var pRaw) || pRaw is null) return null;
                var p = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pRaw));

                if (p is null || !p.TryGetValue("processData", out var pdRaw) || pdRaw is null) return null;
                var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pdRaw));

                if (pd is null || !pd.TryGetValue("level1", out var lRaw) || lRaw is null) return null;
                var level1 = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(lRaw));

                if (level1 is null || !level1.TryGetValue("formData", out var fdRaw) || fdRaw is null) return null;
                var form = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(fdRaw));

                if (form is null || !form.TryGetValue("execution", out var exRaw) || exRaw is null) return null;
                return JsonSerializer.Deserialize<string>(JsonSerializer.Serialize(exRaw));
            }
            catch
            {
                return null;
            }
        }
    }
}
