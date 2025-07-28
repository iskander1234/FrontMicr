using System.Text.Json;
using System.Threading;
using AutoMapper;
using BpmBaseApi.Domain.Entities.Event.Common;
using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Persistence;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Models.Process;
using BpmBaseApi.Shared.Responses.Process;
using Microsoft.Extensions.Caching.Memory;

namespace BpmBaseApi.Services.Implementations
{
    public class ProcessTaskService(
    IUnitOfWork unitOfWork,
    IMapper mapper,
    IMemoryCache cache) : IProcessTaskService
    {
        public async Task HandleDelegationAsync(
        ProcessTaskEntity currentTask,
        ProcessDataEntity processData,
        SendProcessCommand command,
        List<UserInfo> recipients,
        string? comment,
        CancellationToken cancellationToken)
        {
            currentTask.Status = "Waiting";
            await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
            {
                EntityId = currentTask.Id,
                Status = "Waiting"
            }, cancellationToken);

            var payloadJson = JsonSerializer.Serialize(command.PayloadJson);

            foreach (var assignee in recipients)
            {
                await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
                {
                    ProcessDataId = processData.Id,
                    BlockId = currentTask.BlockId,
                    BlockCode = currentTask.BlockCode,
                    BlockName = currentTask.BlockName,
                    AssigneeCode = assignee.UserCode,
                    AssigneeName = assignee.UserName,
                    Status = "Pending",
                    PayloadJson = payloadJson,
                    ParentTaskId = currentTask.Id,
                    ProcessCode = processData.ProcessCode,
                    ProcessName = processData.ProcessName,
                    RegNumber = processData.RegNumber,
                    Title = processData.Title,
                    InitiatorCode = command.SenderUserCode,
                    InitiatorName = command.SenderUserName,
                    Comment = comment
                }, cancellationToken);

                await RefreshUserTaskCacheAsync(assignee.UserCode, cancellationToken);
            }
        }

        public async Task<BlockEntity?> FindNextBlockAsync(
            Guid processId,
            Guid currentBlockId,
            string action,
            string? condition,
            CancellationToken cancellationToken)
        {
            return await unitOfWork.BlockRepository.GetByFilterAsync(cancellationToken,
                b => b.ProcessId == processId &&
                     b.Id == currentBlockId &&
                     b.Action == action &&
                     (string.IsNullOrEmpty(b.Condition) || b.Condition == condition) &&
                     b.NextBlockId != null);
        }

        public async Task CreateTasksAsync(
            Guid blockId,
            string blockCode,
            string blockName,
            List<UserInfo> recipients,
            ProcessDataEntity processData,
            SendProcessCommand command,
            Guid? parentTaskId,
            CancellationToken cancellationToken)
        {
            var payloadJson = JsonSerializer.Serialize(command.PayloadJson);
            foreach (var recipient in recipients)
            {
                await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
                {
                    ProcessDataId = processData.Id,
                    BlockId = blockId,
                    BlockCode = blockCode,
                    BlockName = blockName,
                    Status = "Pending",
                    ProcessCode = processData.ProcessCode,
                    ProcessName = processData.ProcessName,
                    RegNumber = processData.RegNumber,
                    ParentTaskId = parentTaskId,
                    InitiatorCode = command.SenderUserCode,
                    InitiatorName = command.SenderUserName,
                    AssigneeCode = recipient.UserCode,
                    AssigneeName = recipient.UserName,
                    Title = processData.Title,
                    PayloadJson = payloadJson,
                    Comment = ExtractFromPayload<string>(command.PayloadJson, "comment")
                }, cancellationToken);

                await RefreshUserTaskCacheAsync(recipient.UserCode, cancellationToken);
            }
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

        public async Task<ProcessTaskCreatedEvent?> CreateParentIfNeededAsync(List<UserInfo> recipients, BlockEntity nextBlock, ProcessDataEntity processData, SendProcessCommand command, string? comment, CancellationToken cancellationToken)
        {
            if (recipients.Count <= 1)
                return null;

            var parentEvent = new ProcessTaskCreatedEvent
            {
                ProcessDataId = processData.Id,
                BlockId = nextBlock.Id,
                BlockCode = nextBlock.BlockCode,
                BlockName = nextBlock.BlockName,
                AssigneeCode = "system",
                AssigneeName = "system",
                Status = "Waiting",
                ProcessCode = processData.ProcessCode,
                ProcessName = processData.ProcessName,
                RegNumber = processData.RegNumber,
                Title = processData.Title,
                InitiatorCode = command.SenderUserCode,
                InitiatorName = command.SenderUserName,
                Comment = comment
            };

            await unitOfWork.ProcessTaskRepository.RaiseEvent(parentEvent, cancellationToken);
            return parentEvent;
        }

        public async Task<BlockInfo?> TryHandleParentTaskAsync(
    ProcessTaskEntity currentTask,
    BlockEntity currentBlock,
    BlockEntity nextBlock,
    ProcessDataEntity processData,
    SendProcessCommand command,
    string? comment,
    List<UserInfo> recipients,
    CancellationToken cancellationToken)
        {
            if (currentTask.ParentTaskId == null)
                return null;

            var pendingSiblings = await unitOfWork.ProcessTaskRepository.CountAsync(cancellationToken,
                t => t.ParentTaskId == currentTask.ParentTaskId && t.Id != currentTask.Id /*&& t.Status == "Pending"*/);

            if (pendingSiblings > 0)
            {
                return new BlockInfo
                {
                    Handled = true,
                    BlockCode = currentBlock.BlockCode,
                    BlockName = currentBlock.BlockName
                };
            }

            var parentTask = await unitOfWork.ProcessTaskRepository.GetByIdAsync(cancellationToken, currentTask.ParentTaskId.Value);
            if (parentTask == null)
                return null;

            if (parentTask.AssigneeCode == "system")
            {
                await FinalizeTaskAsync(parentTask, cancellationToken);

                var parentCreated = await CreateParentIfNeededAsync(recipients, nextBlock, processData, command, comment, cancellationToken);
                await CreateTasksAsync(nextBlock.Id, nextBlock.BlockCode, nextBlock.BlockName, recipients, processData, command, parentCreated?.EntityId, cancellationToken);

                return new BlockInfo
                {
                    Handled = true,
                    BlockCode = nextBlock.BlockCode,
                    BlockName = nextBlock.BlockName
                };
            }

            await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
            {
                EntityId = parentTask.Id,
                Status = "Pending"
            }, cancellationToken);

            return new BlockInfo
            {
                Handled = true,
                BlockCode = currentBlock.BlockCode,
                BlockName = currentBlock.BlockName
            };
        }


        public BaseResponseDto<SendProcessResponse> SuccessResponse(string blockCode, string blockName) =>
            new()
            {
                Data = new SendProcessResponse
                {
                    BlockCode = blockCode,
                    BlockName = blockName
                }
            };



        private static readonly Dictionary<string, string> ProcessPrefixes = new()
    {
        { "QuarterlyPlanning", "QP" },
        { "AnnualPlanning", "AP" },
        // можно добавить другие процессы
    };
        public async Task<string> GenerateRequestNumberAsync(string processCode, CancellationToken cancellationToken)
        {
            int lastNumber = 1;
            var year = DateTime.Now.Year;
            var counter = await unitOfWork.RequestNumberCounterRepository
                .GetByFilterAsync(CancellationToken.None, p => p.ProcessCode == processCode && p.Year == year);

            if (counter == null)
            {
                var requestNumberCounter = new RequestNumberCounterCreatedEvent
                {
                    ProcessCode = processCode,
                    LastNumber = lastNumber,
                    Year = year
                };
                await unitOfWork.RequestNumberCounterRepository.RaiseEvent(requestNumberCounter, cancellationToken);
            }
            else
            {
                lastNumber = counter.LastNumber + 1;

                var requestNumberCounter = new RequestNumberCounterChangedEvent
                {
                    EntityId = counter.Id,
                    LastNumber = lastNumber,
                };
                await unitOfWork.RequestNumberCounterRepository.RaiseEvent(requestNumberCounter, cancellationToken);
            }

            var prefix = ProcessPrefixes.TryGetValue(processCode, out var value)
            ? value
            : processCode[..2].ToUpper(); // по умолчанию первые 2 буквы

            return $"{prefix}-{lastNumber:D4}-{year}";
        }
        public async Task FinalizeTaskAsync(ProcessTaskEntity task, CancellationToken cancellationToken)
        {
            await unitOfWork.ProcessTaskRepository.DeleteByIdAsync(task.Id, cancellationToken);
            await RefreshUserTaskCacheAsync(task.AssigneeCode, cancellationToken);
        }


        public async Task RefreshUserTaskCacheAsync(string userCode, CancellationToken cancellationToken)
        {
            var updatedTasks = await unitOfWork.ProcessTaskRepository
                .GetByFilterListAsync(cancellationToken, p => p.AssigneeCode.ToLower() == userCode.ToLower());

            var mapped = mapper.Map<List<GetUserTasksResponse>>(updatedTasks);
            cache.Set($"tasks:{userCode}", mapped, TimeSpan.FromHours(3));
        }

        public async Task LogTaskHistoryAsync(Guid processDataId, Guid taskId, string action, string initiator, string assignee, string blockName, string payloadJson, CancellationToken cancellationToken)
        {
            var logEvent = new ProcessTaskHistoryCreatedEvent
            {
                ProcessDataId = processDataId,
                TaskId = taskId,
                Action = action,
                InitiatorCode = initiator,
                AssigneeCode = assignee,
                BlockName = blockName,
                Description = "",
                Timestamp = DateTime.UtcNow,
                PayloadJson = payloadJson
            };

            await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(logEvent, cancellationToken);
        }

        public Task<List<UserInfo>> ResolveRecipientsAsync(string processCode, string blockCode, List<UserInfo> currentRecipients, CancellationToken cancellationToken)
        {
            // Клонируем входной список
            var result = new List<UserInfo>(currentRecipients);

            if (processCode == "Activity" && blockCode == "CoExecutors")
            {
                result = new List<UserInfo>
                {
                    new() { UserCode = "m.ilespayev", UserName = "Меииржан" },
                    new() { UserCode = "l.iskender", UserName = "Лесхан" }
                };
            }

            if (blockCode == "Completion")
            {
                result.Clear(); // Завершение — без получателей
            }

            return Task.FromResult(result);
        }

        public async Task LogHistoryAsync(ProcessTaskEntity task, SendProcessCommand command, string? assignee, CancellationToken cancellationToken)
        {
            var eventPayload = new ProcessTaskHistoryCreatedEvent
            {
                ProcessDataId = task.ProcessDataId,
                TaskId = task.Id,
                ParentTaskId = task.ParentTaskId,
                Action = command.Action.ToString(),
                AssigneeCode = task.AssigneeCode,
                AssigneeName = task.AssigneeName,
                BlockCode = task.BlockCode,
                BlockName = task.BlockName,
                Timestamp = DateTime.Now,
                PayloadJson = JsonSerializer.Serialize(command.PayloadJson),
                Description = "",
                ProcessCode = task.ProcessCode,
                ProcessName = task.ProcessName,
                Comment = task.Comment,
                RegNumber = task.RegNumber,
                Title = task.Title,
                InitiatorCode = task.InitiatorCode,
                InitiatorName = task.InitiatorName
            };

            await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(eventPayload, cancellationToken);
        }
    }
}
