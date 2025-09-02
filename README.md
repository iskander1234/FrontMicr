using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Enum;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using System.Text.Json;
using AutoMapper;
using Microsoft.Extensions.Caching.Memory;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Models.Process;
using BpmBaseApi.Shared.Models.Camunda;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    public class SendProcessCommandHandler(
        IMapper mapper,
        IUnitOfWork unitOfWork,
        IMemoryCache cache,
        ICamundaService camundaService,
        IProcessTaskService processTaskService)
        : IRequestHandler<SendProcessCommand, BaseResponseDto<SendProcessResponse>>
    {
        public async Task<BaseResponseDto<SendProcessResponse>> Handle(SendProcessCommand command,
            CancellationToken cancellationToken)
        {
            // Берём задачу без фильтра по статусу — чтобы поймать Waiting и вернуть 400 с подсказкой
            var currentTask = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                cancellationToken,
                p => p.Id == command.TaskId
            ) ?? throw new HandlerException("Задача не найдена", ErrorCodesEnum.Business);

            var processData =
                await unitOfWork.ProcessDataRepository.GetByIdAsync(cancellationToken, currentTask.ProcessDataId)
                ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

            // Последовательный ли режим из processData.approvalTypeCode
            var pdPayload = JsonSerializer.Deserialize<Dictionary<string, object>>(processData.PayloadJson);
            bool isSequential = ReadIsSequentialFromProcessData(pdPayload);

            // Если Sequentially и задача НЕ Pending → 400 и сказать кто Pending
            if (isSequential && !string.Equals(currentTask.Status, "Pending", StringComparison.OrdinalIgnoreCase))
            {
                ProcessTaskEntity? activePending = null;

                if (currentTask.ParentTaskId != null)
                {
                    activePending = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                        cancellationToken,
                        t => t.ParentTaskId == currentTask.ParentTaskId && t.Status == "Pending"
                    );
                }

                if (activePending == null)
                {
                    activePending = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                        cancellationToken,
                        t => t.ProcessDataId == currentTask.ProcessDataId
                             && t.BlockCode == currentTask.BlockCode
                             && t.Status == "Pending"
                    );
                }

                var responsible = activePending?.AssigneeCode ?? "не найден";
                throw new HandlerException(
                    $"Отправка запрещена: сначала должен отправить исполнитель со статусом Pending ({responsible}).",
                    ErrorCodesEnum.Business // у тебя это маппится в HTTP 400
                );
            }

            // Старая проверка «только Pending» (для параллельного режима как было)
            if (!string.Equals(currentTask.Status, "Pending", StringComparison.OrdinalIgnoreCase))
                throw new HandlerException("Задача не найдена или уже обработана", ErrorCodesEnum.Business);

            var recipients = ExtractFromPayload<List<UserInfo>>(command.PayloadJson, "recipients") ?? new();
            //var comment = ExtractFromPayload<string>(command.PayloadJson, "comment");

            // Делегирование
            if (command.Action == ProcessAction.Delegate && recipients.Any())
            {
                await processTaskService.HandleDelegationAsync(currentTask, processData, command, recipients,
                    cancellationToken);
                return new BaseResponseDto<SendProcessResponse>
                {
                    Data = new SendProcessResponse { Success = true }
                };
            }

            // Если sequential — закрываем текущего. Если negative → чистим остальных и идём к Camunda.
            // Если positive (accept) → поднимаем следующего Waiting -> Pending и выходим.
            if (isSequential)
            {
                if (!Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var stage))
                    throw new InvalidOperationException($"Unknown process stage: {currentTask.BlockCode}");
                
                var parent = await unitOfWork.ProcessTaskRepository
                    .GetByIdAsync(cancellationToken, currentTask.ParentTaskId!.Value);

                var siblings = await unitOfWork.ProcessTaskRepository
                    .GetByFilterListAsync(cancellationToken, t => t.ParentTaskId == parent.Id);

                await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode,
                    cancellationToken);
                await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

                // Был ли отрицательный исход на текущем шаге?
                bool isNegative = stage switch
                {
                    ProcessStage.Rework   => command.Action == ProcessAction.Cancel,
                    ProcessStage.Signing  => command.Action == ProcessAction.Reject,
                    _ /* Approval и прочее */ => command.Condition is ProcessCondition.remake or ProcessCondition.reject
                };


                if (isNegative)
                {
                    
                    var affectedAssignees = siblings
                        .Where(t => t.Status == "Waiting" && !string.IsNullOrEmpty(t.AssigneeCode))
                        .Select(t => t.AssigneeCode)
                        .Distinct()
                        .ToList();

                    foreach (var w in siblings.Where(t => t.Status == "Waiting"))
                    {
                        await processTaskService.FinalizeTaskAsync(w, cancellationToken);
                    }

                    // у тех, у кого были waiting, теперь ничего не должно висеть
                    foreach (var a in affectedAssignees)
                        await processTaskService.RefreshUserTaskCacheAsync(a, cancellationToken);
                    // 1) Останавливаем цепочку: закрываем ВСЕ оставшиеся Waiting под тем же родителем
                    // foreach (var w in siblings.Where(t => t.Status == "Waiting"))
                    // {
                    //     await processTaskService.FinalizeTaskAsync(w, cancellationToken);
                    // }

                    // 2) Не поднимаем следующего. Дальше код пойдёт к parentTask (Camunda) и отправит false.
                    // Просто выходим из этого if, НИЧЕГО не возвращаем здесь.
                }
                else
                {
                    // accept: продолжаем цепочку
                    var next = siblings
                        .Where(t => t.Status == "Waiting")
                        .OrderBy(t => t.Order ?? int.MaxValue)
                        .ThenBy(t => t.Created)
                        .FirstOrDefault();

                    if (next != null)
                    {
                        await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
                        {
                            EntityId = next.Id,
                            Status = "Pending"
                        }, cancellationToken);

                        // у next теперь новая Pending — обновим его кэш
                        if (!string.IsNullOrEmpty(next.AssigneeCode))
                            await processTaskService.RefreshUserTaskCacheAsync(next.AssigneeCode, cancellationToken);

                        return new BaseResponseDto<SendProcessResponse>
                        {
                            Data = new SendProcessResponse { Success = true }
                        };
                    }

                    // сюда попадём, если очередников не осталось — пойдём сабмитить родителя как обычно
                }
            }


             var pendingSiblings = await unitOfWork.ProcessTaskRepository.CountAsync(
                     cancellationToken,
                     t => t.ParentTaskId == currentTask.ParentTaskId
                          && t.Id != currentTask.Id
                          && t.Status == "Pending");

            if (pendingSiblings > 0)
            {
                await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode,
                    cancellationToken);
                await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

                return new BaseResponseDto<SendProcessResponse>
                {
                    Data = new SendProcessResponse { Success = true }
                };
            }

            var parentTask =
                await unitOfWork.ProcessTaskRepository.GetByIdAsync(cancellationToken, currentTask.ParentTaskId.Value);

            if (parentTask.AssigneeCode == "system")
            {
                var claimedTasksResponse =
                    await camundaService.CamundaClaimedTasks(processData.ProcessInstanceId, parentTask.BlockCode)
                    ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);
                
                if (!Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var stageForCamunda))
                    throw new InvalidOperationException($"Unknown process stage: {currentTask.BlockCode}");

                // >>> НОВОЕ: получаем правильные булевы переменные под конкретный этап
                var variables = await BuildCamundaVariablesAsync(
                    stageForCamunda,
                    command.Condition, // accept/remake/reject
                    command.Action, // <<< ДОБАВИЛИ
                    processData.Id,
                    parentTask.Id, // нужен для Approval-раунда
                    unitOfWork,
                    cancellationToken);

                var submitResponse = await camundaService.CamundaSubmitTask(
                    new CamundaSubmitTaskRequest
                    {
                        TaskId = claimedTasksResponse.TaskId,
                        Variables = variables
                    });

                if (!submitResponse.Success)
                {
                    throw new HandlerException($"Ошибка при отправке Msg:{submitResponse.Msg}", ErrorCodesEnum.Camunda);
                }

                await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode,
                    cancellationToken);
                await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
                await processTaskService.FinalizeTaskAsync(parentTask, cancellationToken);

                return new BaseResponseDto<SendProcessResponse>
                {
                    Data = new SendProcessResponse { Success = true }
                };
            }
            
            //Добавил 
            await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
            await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
            
            // обновляем кэш того, кто только что отправил (его Pending исчезла)
            await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);
            
            await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
            {
                EntityId = parentTask.Id,
                Status = "Pending"
            }, cancellationToken);

            // текущего — убрать из кэша его Pending
            await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

            // родителя — добавить ему Pending (если есть реальный исполнитель)
            if (!string.IsNullOrWhiteSpace(parentTask.AssigneeCode) &&
                !string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
            {
                await processTaskService.RefreshUserTaskCacheAsync(parentTask.AssigneeCode, cancellationToken);
            }

            return new BaseResponseDto<SendProcessResponse>
            {
                Data = new SendProcessResponse { Success = true }
            };

        }

     


        private static string GetCamundaVarNameForStage(ProcessStage stage) => stage switch
        {
            ProcessStage.Approval => "agreement",
            ProcessStage.Signing => "sign",
            ProcessStage.Execution => "executed", // если в BPMN ждёшь другое имя — поменяй тут
            ProcessStage.ExecutionCheck => "checked", // при необходимости переименуй
            ProcessStage.Rework => "refinement", // <<< БЫЛО "agreement", теперь "refinement"
            _ => "agreement" // безопасный дефолт
        };

      

        private static async Task<Dictionary<string, object>> BuildCamundaVariablesAsync(
            ProcessStage stage,
            ProcessCondition condition,
            ProcessAction action,
            Guid processDataId,
            Guid? parentTaskId,
            IUnitOfWork unitOfWork,
            CancellationToken ct)
        {
            var varName = GetCamundaVarNameForStage(stage);

            switch (stage)
            {
                case ProcessStage.Rework:
                    // refinement: Submit => true, Cancel => false
                    return new Dictionary<string, object>
                    {
                        [varName] = (action == ProcessAction.Submit)
                    };

                case ProcessStage.Signing:
                    // sign: Submit => true, Reject => false
                    return new Dictionary<string, object>
                    {
                        [varName] = (action == ProcessAction.Submit)
                    };

                case ProcessStage.Approval:
                {
                    // agreement: accept => true, remake/reject => false
                    var isPositive = (condition == ProcessCondition.accept);

                    if (isPositive)
                    {
                        // «вето» в рамках того же родителя (раунда)
                        isPositive = await IsStagePositiveConsideringHistoryAsync(
                            ProcessStage.Approval, isPositive, processDataId, parentTaskId, unitOfWork, ct);
                    }

                    return new Dictionary<string, object> { [varName] = isPositive };
                }

                default:
                {
                    // дефолт: accept => true
                    var isPositive = (condition == ProcessCondition.accept);
                    return new Dictionary<string, object> { [varName] = isPositive };
                }
            }
        }

        /// <summary>
        /// true, если processData.approvalTypeCode == "Sequentially"
        /// </summary>
        private static bool ReadIsSequentialFromProcessData(Dictionary<string, object>? payload)
        {
            if (payload == null) return false;
            try
            {
                if (!payload.TryGetValue("processData", out var pdRaw) || pdRaw is null) return false;

                var json = JsonSerializer.Serialize(pdRaw);
                var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(json);

                if (pd != null && pd.TryGetValue("approvalTypeCode", out var typeRaw) && typeRaw is not null)
                {
                    var tJson = JsonSerializer.Serialize(typeRaw);
                    var type = JsonSerializer.Deserialize<string>(tJson);
                    return string.Equals(type, "Sequentially", StringComparison.OrdinalIgnoreCase);
                }
            }
            catch
            {
                /* считаем Parallel */
            }

            return false;
        }

        private static async Task<bool> IsStagePositiveConsideringHistoryAsync(
            ProcessStage stage,
            bool currentIsPositive,
            Guid processDataId,
            Guid? parentTaskId,
            IUnitOfWork unitOfWork,
            CancellationToken ct)
        {
            if (!currentIsPositive) return false;

            var stageCode = stage.ToString();

            var negativeCount = await unitOfWork.ProcessTaskHistoryRepository.CountAsync(
                ct,
                h => h.ProcessDataId == processDataId
                     && h.BlockCode == stageCode
                     && h.ParentTaskId == parentTaskId
                     && (h.Condition == nameof(ProcessCondition.remake)
                         || h.Condition == nameof(ProcessCondition.reject)));

            return negativeCount == 0;
        }

     
        public static T? ExtractFromPayload<T>(Dictionary<string, object>? payload, string key)
        {
            if (payload == null || !payload.TryGetValue(key, out var value))
                return default;

            var json = JsonSerializer.Serialize(value);
            return JsonSerializer.Deserialize<T>(json, new JsonSerializerOptions
            {
                PropertyNameCaseInsensitive = true
            });
        }
    }
}


