var parentTask =
    await unitOfWork.ProcessTaskRepository.GetByIdAsync(cancellationToken, currentTask.ParentTaskId.Value);

if (parentTask.AssigneeCode == "system")
{
    // ... тут всё ок: ты сабмитишь в Camunda и ПОТОМ делаешь
    // LogHistoryAsync(currentTask) + FinalizeTaskAsync(currentTask) + FinalizeTaskAsync(parentTask)
    return Success;
}

// >>> ВОТ ЭТА ВЕТКА ! <<<
await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
{
    EntityId = parentTask.Id,
    Status = "Pending"
}, cancellationToken);

return Success;




// ... выше код

var parentTask =
    await unitOfWork.ProcessTaskRepository.GetByIdAsync(cancellationToken, currentTask.ParentTaskId.Value);

if (parentTask.AssigneeCode == "system")
{
    // всё как у тебя было — ок
    // LogHistory + Finalize(currentTask) + Finalize(parentTask)
    // return
}

// <-- ДОБАВЬ ЭТО:
await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

// поднимаем родителя
await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
{
    EntityId = parentTask.Id,
    Status = "Pending"
}, cancellationToken);

return new BaseResponseDto<SendProcessResponse>
{
    Data = new SendProcessResponse { Success = true }
};
