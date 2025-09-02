Sequential ветка — после FinalizeTaskAsync(currentTask) и подъёма next:

await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

// ОБНОВИТЬ КЭШ текущего пользователя (его Pending ушла)
await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

// если был отрицательный исход — мы финализируем waiting-siblings
if (isNegative)
{
    var affectedAssignees = siblings
        .Where(t => t.Status == "Waiting" && !string.IsNullOrEmpty(t.AssigneeCode))
        .Select(t => t.AssigneeCode)
        .Distinct()
        .ToList();

    foreach (var w in siblings.Where(t => t.Status == "Waiting"))
        await processTaskService.FinalizeTaskAsync(w, cancellationToken);

    // ОБНОВИТЬ КЭШ для всех, у кого были waiting (теперь они исчезли)
    foreach (var a in affectedAssignees)
        await processTaskService.RefreshUserTaskCacheAsync(a, cancellationToken);
}
else
{
    // подняли следующего исполнителя
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

        // ОБНОВИТЬ КЭШ для следующего исполнителя (у него появилась Pending)
        if (!string.IsNullOrEmpty(next.AssigneeCode))
            await processTaskService.RefreshUserTaskCacheAsync(next.AssigneeCode, cancellationToken);

        return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };
    }
}




Параллельный режим — когда есть pendingSiblings > 0

await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

// ОБНОВИТЬ КЭШ текущего пользователя
await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };

Camunda-ветка (parent = system)

После финализации currentTask и parentTask — обновляем кэш у текущего исполнителя.

await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
await processTaskService.FinalizeTaskAsync(parentTask, cancellationToken);

// ОБНОВИТЬ КЭШ текущего пользователя
await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };


Обычная ветка (parent != system) — переводим родителя в Pending
await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
{
    EntityId = parentTask.Id,
    Status = "Pending"
}, cancellationToken);

// ОБНОВИТЬ КЭШ текущего пользователя (его задача закрыта)
await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

// ОБНОВИТЬ КЭШ родителя, если он не system и есть реальный assignee
if (!string.IsNullOrWhiteSpace(parentTask.AssigneeCode) &&
    !string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
{
    await processTaskService.RefreshUserTaskCacheAsync(parentTask.AssigneeCode, cancellationToken);
}
