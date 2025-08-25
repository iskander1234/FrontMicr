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

            // Если sequential — закрываем текущего, поднимаем следующего Waiting -> Pending по Order, затем Created
            if (isSequential)
            {
                var parent = await unitOfWork.ProcessTaskRepository
                    .GetByIdAsync(cancellationToken, currentTask.ParentTaskId!.Value);

                var siblings = await unitOfWork.ProcessTaskRepository
                    .GetByFilterListAsync(cancellationToken, t => t.ParentTaskId == parent.Id);

                await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode,
                    cancellationToken);
                await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

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

                    return new BaseResponseDto<SendProcessResponse>
                    {
                        Data = new SendProcessResponse { Success = true }
                    };
                }

                // Очередь исчерпана — ниже пошла твоя старая логика с Camunda/родителем
            }

            var pendingSiblings = await unitOfWork.ProcessTaskRepository.CountAsync(cancellationToken,
                t => t.ParentTaskId == currentTask.ParentTaskId && t.Id != currentTask.Id /*&& t.Status == "Pending"*/);

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

                if (!Enum.TryParse<ProcessStage>(parentTask.BlockCode, out var stage))
                {
                    throw new InvalidOperationException($"Unknown process stage: {parentTask.BlockCode}");
                }

                // >>> НОВОЕ: получаем правильные булевы переменные под конкретный этап
                var variables = await BuildCamundaVariablesAsync(
                    stage,
                    command.Condition,         // accept/remake/reject
                    processData.Id,
                    parentTask.Id,             // нужен для Approval-раунда
                    unitOfWork,
                    cancellationToken);
                
                var submitResponse = await camundaService.CamundaSubmitTask(
                    new CamundaSubmitTaskRequest
                    {
                        TaskId    = claimedTasksResponse.TaskId,
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

            await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
            {
                EntityId = parentTask.Id,
                Status = "Pending"
            }, cancellationToken);

            return new BaseResponseDto<SendProcessResponse>
            {
                Data = new SendProcessResponse { Success = true }
            };
        }

        private Dictionary<string, object> GetCamundaVariablesForStage(ProcessStage stage)
        {
            return stage switch
            {
                ProcessStage.Approval => new() { { "agreement", true } },
                ProcessStage.Signing => new() { { "sign", true } },
                _ => new()
            };
        }


        private static string GetCamundaVarNameForStage(ProcessStage stage) => stage switch
        {
            ProcessStage.Approval => "agreement",
            ProcessStage.Signing => "sign",
            ProcessStage.Execution => "executed", // если в BPMN ждёшь другое имя — поменяй тут
            ProcessStage.ExecutionCheck => "checked", // при необходимости переименуй
            ProcessStage.Rework => "agreement", // обычно на этот этап не сабмитят, но оставим дефолт
            _ => "agreement" // безопасный дефолт
        };

        private static bool IsPositiveByCondition(ProcessCondition condition) =>
            condition == ProcessCondition.accept;

        // Для Approval учитываем коллективное «вето»
        private static async Task<bool> IsApprovalPositiveConsideringHistoryAsync(
            bool currentIsPositive,
            Guid processDataId,
            Guid? parentTaskId,
            IUnitOfWork unitOfWork,
            CancellationToken ct)
        {
            if (!currentIsPositive) return false; // уже отрицательно — дальше не проверяем

            // Ищем в истории по этому же раунду (ParentTaskId) любой remake/reject
            var negativeCount = await unitOfWork.ProcessTaskHistoryRepository.CountAsync(
                ct,
                h => h.ProcessDataId == processDataId
                     && h.BlockCode == ProcessStage.Approval.ToString()
                     && h.ParentTaskId == parentTaskId
                     && (h.Condition == nameof(ProcessCondition.remake)
                         || h.Condition == nameof(ProcessCondition.reject)));

            return negativeCount == 0; // если были отрицательные — итог false
        }

        private static async Task<Dictionary<string, object>> BuildCamundaVariablesAsync(
            ProcessStage stage,
            ProcessCondition condition,
            Guid processDataId,
            Guid? parentTaskId,
            IUnitOfWork unitOfWork,
            CancellationToken ct)
        {
            var varName = GetCamundaVarNameForStage(stage);

            bool isPositive = IsPositiveByCondition(condition);

            if (stage == ProcessStage.Approval)
                isPositive = await IsApprovalPositiveConsideringHistoryAsync(
                    isPositive, processDataId, parentTaskId, unitOfWork, ct);

            // На других этапах (Signing/Execution/…) просто маппим accept→true, remake/reject→false
            return new Dictionary<string, object> { [varName] = isPositive };
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

        /*
         public async Task<BaseResponseDto<SendProcessResponse>> Handle(SendProcessCommand command, CancellationToken cancellationToken)
        {
            var currentTask = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(cancellationToken,
                p => p.Id == command.TaskId &&
        p.Status == "Pending")
                ?? throw new HandlerException("Задача не найдена или уже обработана", ErrorCodesEnum.Business);

        var processData = await unitOfWork.ProcessDataRepository.GetByIdAsync(cancellationToken, currentTask.ProcessDataId)
            ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

        var recipients = ExtractFromPayload<List<UserInfo>>(command.PayloadJson, "recipients") ?? new();
        var comment = ExtractFromPayload<string>(command.PayloadJson, "comment");


            // ===== Делегирование =====
            if (command.Action == ProcessAction.Delegate && recipients.Any())
            {
                await processTaskService.HandleDelegationAsync(currentTask, processData, command, recipients, comment, cancellationToken);
                //await processTaskService.LogHistoryAsync(currentTask, command, "", cancellationToken);
                return processTaskService.SuccessResponse(currentTask.BlockCode, currentTask.BlockName);
            }

            // ===== Переход к следующему блоку =====
            var currentBlock = await processTaskService.FindNextBlockAsync(
                processData.ProcessId,
                currentTask.BlockId,
     rte           command.Action.ToString(),
        command.Condition?.ToString(),
        cancellationToken
            ) ?? throw new HandlerException("Следующий блок не найден", ErrorCodesEnum.Business);

        var nextBlock = await unitOfWork.BlockRepository.GetByFilterAsync(cancellationToken,
            b => b.ProcessId == processData.ProcessId && b.Id == currentBlock.NextBlockId)
            ?? throw new HandlerException("Подробности следующего блока не найдены", ErrorCodesEnum.Business);


        // Завершение текущей подзадачи и проверка родителя
        await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
        await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
            //await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

            recipients = await processTaskService.ResolveRecipientsAsync(processData.ProcessCode, nextBlock.BlockCode, recipients, cancellationToken);

    var blockResult = await processTaskService.TryHandleParentTaskAsync(currentTask, currentBlock, nextBlock, processData, command, comment, recipients, cancellationToken);

            if (blockResult?.Handled == true)
            {
                if (blockResult.BlockCode == "Completion")
                {
                    await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataStatusChangedEvent
                    {
                        EntityId = processData.Id,
                        StatusCode = "Completed",
                        StatusName = "Завершено"
                    }, cancellationToken);
                }
                await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataBlockChangedEvent
                {
                    EntityId = processData.Id,
                    BlockCode = blockResult.BlockCode,
                    BlockName = blockResult.BlockName
                }, cancellationToken);
return processTaskService.SuccessResponse(blockResult.BlockCode, blockResult.BlockName);
            }

            var parentCreatedEvent = await processTaskService.CreateParentIfNeededAsync(recipients, nextBlock, processData, command, comment, cancellationToken);
await processTaskService.CreateTasksAsync(nextBlock.Id, nextBlock.BlockCode, nextBlock.BlockName, recipients, processData, command, parentCreatedEvent?.EntityId, cancellationToken);

await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataBlockChangedEvent
{
    EntityId = processData.Id,
    BlockCode = nextBlock.BlockCode,
    BlockName = nextBlock.BlockName
}, cancellationToken);

return processTaskService.SuccessResponse(nextBlock.BlockCode, nextBlock.BlockName);
        }
         */
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

        private static bool TryReadSequenceFlag(Dictionary<string, object>? payload)
        {
            if (payload == null) return false;
            try
            {
                if (!payload.TryGetValue("sysInfo", out var sysInfoRaw) || sysInfoRaw == null)
                    return false;

                var json = JsonSerializer.Serialize(sysInfoRaw);
                var sysInfo = JsonSerializer.Deserialize<Dictionary<string, object>>(json);

                if (sysInfo != null && sysInfo.TryGetValue("sequence", out var seqRaw))
                {
                    var seqJson = JsonSerializer.Serialize(seqRaw);
                    var seq = JsonSerializer.Deserialize<bool?>(seqJson);
                    return seq == true;
                }
            }
            catch
            {
                /* игнор, считаем false */
            }

            return false;
        }
    }
}


