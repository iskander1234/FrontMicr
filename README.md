// 3) Собираем переменные для Camunda (может быть пусто)
var variables = BuildItsmLevelVariables(command.PayloadJson);

// >>> NEW: если есть что прокинуть — кладём их в процесс ДО ветвления по system
if (variables.Count > 0)
{
    // Нужен метод в ICamundaService, который сетит vars на уровне process instance
    await camundaService.CamundaSetProcessVariables(processData.ProcessInstanceId, variables);
}

// 4) Если родительская Camunda-задача на system — сабмитим её (переменные можно не дублировать,
//   но можно и передать ещё раз — не повредит)
if (string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
{
    var claimed = await camundaService.CamundaClaimedTasks(
                      processData.ProcessInstanceId, parentTask.BlockCode)
                  ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);

    var submit = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
    {
        TaskId = claimed.TaskId,
        Variables = variables // можно оставить пустым — мы уже установили их выше
    });

    if (!submit.Success)
        throw new HandlerException($"Ошибка при отправке Msg:{submit.Msg}", ErrorCodesEnum.Camunda);

    await processTaskService.FinalizeTaskAsync(parentTask, ct);
}
else
{
    // как и было
    await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
    {
        EntityId = parentTask.Id,
        Status = "Pending"
    }, ct);

    if (!string.IsNullOrWhiteSpace(parentTask.AssigneeCode))
        await processTaskService.RefreshUserTaskCacheAsync(parentTask.AssigneeCode, ct);
}
