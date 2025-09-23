if (string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
{
    var claimed = await camundaService.CamundaClaimedTasks(
                      processData.ProcessInstanceId, parentTask.BlockCode)
                  ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);

    var variables = new Dictionary<string, object>
    {
        ["classification"] = classification
    };

    var submit = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
    {
        TaskId    = claimed.TaskId,
        Variables = variables
    });

    if (!submit.Success)
        throw new HandlerException($"Ошибка при отправке Msg:{submit.Msg}", ErrorCodesEnum.Camunda);

    // <-- ВАЖНО: сначала лог и финализация текущей (дочерней) задачи
    await processTaskService.LogHistoryItsmAsync(currentTask, command, currentTask.AssigneeCode, ct);
    await processTaskService.FinalizeTaskAsync(currentTask, ct);
    await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, ct);

    // <-- Затем финализируем родителя (system)
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











var others = await unitOfWork.ProcessTaskRepository
    .GetByFilterListAsync(ct, t => t.ParentTaskId == parentTask.Id && t.Id != currentTask.Id);

foreach (var ch in others)
    await processTaskService.FinalizeTaskAsync(ch, ct);

// затем удаляйте parentTask
await processTaskService.FinalizeTaskAsync(parentTask, ct);
