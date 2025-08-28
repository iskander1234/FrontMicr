var alreadyLoggedAndFinalized = false;

if (isSequential)
{
    if (currentTask.ParentTaskId is null)
        throw new HandlerException("У задачи не указан родитель", ErrorCodesEnum.Business);

    var parent = await unitOfWork.ProcessTaskRepository.GetByIdAsync(cancellationToken, currentTask.ParentTaskId.Value);
    if (!Enum.TryParse<ProcessStage>(parent.BlockCode, out var stage))
        throw new InvalidOperationException($"Unknown process stage: {parent.BlockCode}");

    var siblings = await unitOfWork.ProcessTaskRepository
        .GetByFilterListAsync(cancellationToken, t => t.ParentTaskId == parent.Id);

    await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
    await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
    alreadyLoggedAndFinalized = true;

    bool isNegative = stage switch
    {
        ProcessStage.Rework  => command.Action == ProcessAction.Cancel,
        ProcessStage.Signing => command.Action == ProcessAction.Reject,
        _ => command.Condition is ProcessCondition.remake or ProcessCondition.reject
    };

    if (isNegative)
    {
        foreach (var s in siblings.Where(t => t.Status == "Waiting" || t.Status == "Pending"))
            await processTaskService.FinalizeTaskAsync(s, cancellationToken);
        // падаем дальше к Camunda
    }
    else
    {
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
        // если «очередников» больше нет — идём к Camunda
    }
}
