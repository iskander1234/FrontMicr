Б
await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

var affectedAssignees = siblings
    .Where(t => t.Status == "Waiting" && !string.IsNullOrEmpty(t.AssigneeCode))
    .Select(t => t.AssigneeCode)
    .Distinct()
    .ToList();

foreach (var w in siblings.Where(t => t.Status == "Waiting"))
{
    await processTaskService.FinalizeTaskAsync(w, cancellationToken);
}

foreach (var a in affectedAssignees)
    await processTaskService.RefreshUserTaskCacheAsync(a, cancellationToken);
С

await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

// текущего — убрать из кэша его Pending
await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

var affectedAssignees = siblings
    .Where(t => t.Status == "Waiting" && !string.IsNullOrEmpty(t.AssigneeCode))
    .Select(t => t.AssigneeCode)
    .Distinct()
    .ToList();

foreach (var w in siblings.Where(t => t.Status == "Waiting"))
{
    await processTaskService.FinalizeTaskAsync(w, cancellationToken);
}

// у тех, у кого были waiting — почистить
foreach (var a in affectedAssignees)
    await processTaskService.RefreshUserTaskCacheAsync(a, cancellationToken);





Б
await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

return new BaseResponseDto<SendProcessResponse>
{
    Data = new SendProcessResponse { Success = true }
};

с
await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

// текущего — убрать из кэша его Pending
await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

return new BaseResponseDto<SendProcessResponse>
{
    Data = new SendProcessResponse { Success = true }
};


Б
// обновляем кэш того, кто только что отправил (его Pending исчезла)
await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
{
    EntityId = parentTask.Id,
    Status = "Pending"
}, cancellationToken);

// текущего — убрать из кэша его Pending
await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

// родителя — добавить ему Pending ...
if (!string.IsNullOrWhiteSpace(parentTask.AssigneeCode) &&
    !string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
{
    await processTaskService.RefreshUserTaskCacheAsync(parentTask.AssigneeCode, cancellationToken);
}
с
// финализировали current выше — теперь обновим его кэш один раз
await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

// поднимем родителя
await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
{
    EntityId = parentTask.Id,
    Status = "Pending"
}, cancellationToken);

// у родителя появился Pending — обновим его кэш (если реальный исполнитель)
if (!string.IsNullOrWhiteSpace(parentTask.AssigneeCode) &&
    !string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
{
    await processTaskService.RefreshUserTaskCacheAsync(parentTask.AssigneeCode, cancellationToken);
}
