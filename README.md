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
    /// - НЕ требует classification для этого маршрута
    /// - при системном parent: сабмитим Camunda и отправляем переменные Line 1/2/3 по правилам
    /// - иначе поднимаем родителя в Pending
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
            // 1) Текущая подзадача должна быть Pending
            var currentTask = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                                  ct, t => t.Id == command.TaskId && t.Status == "Pending")
                              ?? throw new HandlerException("Задача не найдена или уже обработана",
                                  ErrorCodesEnum.Business);

            var processData = await unitOfWork.ProcessDataRepository.GetByIdAsync(ct, currentTask.ProcessDataId)
                              ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

            var parentTask = await unitOfWork.ProcessTaskRepository.GetByIdAsync(ct, currentTask.ParentTaskId!.Value);

            if (string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
            {
                // Для Camunda используем сопоставление BlockCode -> фактический ключ/имя user-task
                var camundaStage = MapToCamundaStageKey(parentTask.BlockCode);

                var claimed = await camundaService.CamundaClaimedTasks(
                                  processData.ProcessInstanceId, camundaStage, ct)
                              ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);

                // Переменные процесса (минимум). classification не обязателен в этой ветке.
                var variables = new Dictionary<string, object>();

                // Если всё-таки пришла classification — передадим (не обязательно)
                var classification = ReadClassification(command.PayloadJson);
                if (!string.IsNullOrWhiteSpace(classification))
                    variables["classification"] = classification!;

                // Логика для Level1/Level2/Level3
                if (Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var stage))
                {
                    switch (stage)
                    {
                        case ProcessStage.Level1:
                        {
                            // executed => true, toSecondLine => false
                            var execution = ReadExecution(command.PayloadJson); // "executed" | "toSecondLine" | null
                            bool line1 = string.Equals(execution, "executed", StringComparison.OrdinalIgnoreCase)
                                ? true
                                : string.Equals(execution, "toSecondLine", StringComparison.OrdinalIgnoreCase)
                                    ? false
                                    : true; // дефолт: true
                            variables["Line 1"] = line1;
                            break;
                        }
                        case ProcessStage.Level2:
                        {
                            // ожидаем bool в payload->processData->level2->formData->line2
                            bool line2 = ReadLineBool(command.PayloadJson, 2) ?? true;
                            variables["Line 2"] = line2;
                            break;
                        }
                        case ProcessStage.Level3:
                        {
                            // ожидаем bool в payload->processData->level3->formData->line3
                            bool line3 = ReadLineBool(command.PayloadJson, 3) ?? true;
                            variables["Line 3"] = line3;
                            break;
                        }
                    }
                }

                var submit = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
                {
                    TaskId    = claimed.TaskId,
                    Variables = variables
                }, ct);

                if (!submit.Success)
                    throw new HandlerException($"Ошибка при отправке Msg:{submit.Msg}", ErrorCodesEnum.Camunda);

                // Сначала лог и финализация текущей (дочерней) задачи
                await processTaskService.LogHistoryItsmAsync(currentTask, command, currentTask.AssigneeCode, ct);
                await processTaskService.FinalizeTaskAsync(currentTask, ct);
                await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, ct);

                // Затем финализируем родителя (system)
                await processTaskService.FinalizeTaskAsync(parentTask, ct);
            }
            else
            {
                // Родительский таск не системный: переводим его в Pending
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

            return new BaseResponseDto<SendProcessResponse>
            {
                Data = new SendProcessResponse { Success = true }
            };
        }

        // === Маппинг вашего BlockCode -> фактический ключ/имя user-task в Camunda ===
        private static string MapToCamundaStageKey(string blockCode)
        {
            // Подставьте реальные taskDefinitionKey (или имя задачи) из BPMN
            return blockCode switch
            {
                "Level1" => "FirstLine",    // например: taskDefinitionKey="FirstLine"
                "Level2" => "SecondLine",
                "Level3" => "ThirdLine",
                _        => blockCode
            };
        }

        // === опционально: classification, если вдруг нужно параллельно с новой схемой ===
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

        // === Level1: читаем execution из payload->processData->level1->formData->execution ===
        private static string? ReadExecution(Dictionary<string, object>? payload)
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

        // === Level2/Level3: читаем bool line2/line3 из payload->processData->levelX->formData->lineX ===
        private static bool? ReadLineBool(Dictionary<string, object>? payload, int levelIndex)
        {
            try
            {
                if (payload is null) return null;

                if (!payload.TryGetValue("payload", out var pRaw) || pRaw is null) return null;
                var p = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pRaw));

                if (p is null || !p.TryGetValue("processData", out var pdRaw) || pdRaw is null) return null;
                var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pdRaw));

                var levelKey = $"level{levelIndex}";
                if (pd is null || !pd.TryGetValue(levelKey, out var lRaw) || lRaw is null) return null;
                var level = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(lRaw));

                if (level is null || !level.TryGetValue("formData", out var fdRaw) || fdRaw is null) return null;
                var form = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(fdRaw));

                var lineKey = $"line{levelIndex}";
                if (form is null || !form.TryGetValue(lineKey, out var lineRaw) || lineRaw is null) return null;

                return JsonSerializer.Deserialize<bool?>(JsonSerializer.Serialize(lineRaw));
            }
            catch
            {
                return null;
            }
        }
    }
}
