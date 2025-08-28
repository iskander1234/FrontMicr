public async Task HandleDelegationAsync(
        ProcessTaskEntity currentTask,
        ProcessDataEntity processData,
        SendProcessCommand command,
        List<UserInfo> recipients,
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
                    Comment = command.Comment
                }, cancellationToken);

                await RefreshUserTaskCacheAsync(assignee.UserCode, cancellationToken);
            }
        }
