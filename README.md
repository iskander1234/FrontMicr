await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
s
await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

// обновляем кэш того, кто только что отправил (его Pending исчезла)
await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);



b
foreach (var w in siblings.Where(t => t.Status == "Waiting"))
{
    await processTaskService.FinalizeTaskAsync(w, cancellationToken);
}
s
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


b
await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
{
    EntityId = next.Id,
    Status = "Pending"
}, cancellationToken);

return new BaseResponseDto<SendProcessResponse>
{
    Data = new SendProcessResponse { Success = true }
};
s
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




b
await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

return new BaseResponseDto<SendProcessResponse>
{
    Data = new SendProcessResponse { Success = true }
};
s
await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

// текущая Pending исчезла у currentTask.AssigneeCode
await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

return new BaseResponseDto<SendProcessResponse>
{
    Data = new SendProcessResponse { Success = true }
};





b
await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
await processTaskService.FinalizeTaskAsync(parentTask, cancellationToken);

return new BaseResponseDto<SendProcessResponse>
{
    Data = new SendProcessResponse { Success = true }
};
s
await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
await processTaskService.FinalizeTaskAsync(parentTask, cancellationToken);

// у текущего исполнителя Pending ушла
await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

return new BaseResponseDto<SendProcessResponse>
{
    Data = new SendProcessResponse { Success = true }
};



b
await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
{
    EntityId = parentTask.Id,
    Status = "Pending"
}, cancellationToken);

return new BaseResponseDto<SendProcessResponse>
{
    Data = new SendProcessResponse { Success = true }
};
s
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


