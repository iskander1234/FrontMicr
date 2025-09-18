Внутри Handle(...) заменяем блок if (isSequential) { ... } целиком на:


if (isSequential)
{
    if (!Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var stage))
        throw new InvalidOperationException($"Unknown process stage: {currentTask.BlockCode}");

    var parent = await unitOfWork.ProcessTaskRepository
        .GetByIdAsync(cancellationToken, currentTask.ParentTaskId!.Value);

    var siblings = await unitOfWork.ProcessTaskRepository
        .GetByFilterListAsync(cancellationToken, t => t.ParentTaskId == parent.Id);

    // 1) лог + закрываем текущего (ОДИН РАЗ)
    await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
    await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
    await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

    // 2) определяем исход шага
    bool isNegative = stage switch
    {
        ProcessStage.Rework  => command.Action == ProcessAction.Cancel,
        ProcessStage.Signing => command.Action == ProcessAction.Reject,
        _                    => command.Condition is ProcessCondition.remake or ProcessCondition.reject
    };

    if (isNegative)
    {
        // закрываем все Waiting в этом раунде
        var affectedAssignees = siblings
            .Where(t => t.Status == "Waiting" && !string.IsNullOrEmpty(t.AssigneeCode))
            .Select(t => t.AssigneeCode)
            .Distinct()
            .ToList();

        foreach (var w in siblings.Where(t => t.Status == "Waiting"))
            await processTaskService.FinalizeTaskAsync(w, cancellationToken);

        foreach (var a in affectedAssignees)
            await processTaskService.RefreshUserTaskCacheAsync(a, cancellationToken);

        // завершаем раунд (Camunda/родитель) и ВОЗВРАЩАЕМСЯ
        await HandleEndOfSequentialRoundAsync(parent, processData, stage, command, cancellationToken);
        return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };
    }
    else
    {
        // positive: ищем следующего
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
                Status   = "Pending"
            }, cancellationToken);

            if (!string.IsNullOrEmpty(next.AssigneeCode))
                await processTaskService.RefreshUserTaskCacheAsync(next.AssigneeCode, cancellationToken);

            return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };
        }

        // «следующего» нет — раунд завершён: Camunda/родитель и ВОЗВРАЩАЕМСЯ
        await HandleEndOfSequentialRoundAsync(parent, processData, stage, command, cancellationToken);
        return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };
    }
}



Ниже по коду оберни «параллельную логику» в if (!isSequential)


var pendingSiblings = await unitOfWork.ProcessTaskRepository.CountAsync(
    cancellationToken,
    t => t.ParentTaskId == currentTask.ParentTaskId
         && t.Id != currentTask.Id
         && t.Status == "Pending");
...



и заканчивается возвратом Success, должен выполняться только когда не sequential. То есть:

if (!isSequential)
{
    var pendingSiblings = await unitOfWork.ProcessTaskRepository.CountAsync(
        cancellationToken,
        t => t.ParentTaskId == currentTask.ParentTaskId
             && t.Id != currentTask.Id
             && t.Status == "Pending");

    if (pendingSiblings > 0)
    {
        await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
        await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
        await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

        return new BaseResponseDto<SendProcessResponse>
        {
            Data = new SendProcessResponse { Success = true }
        };
    }

    var parentTask = await unitOfWork.ProcessTaskRepository
        .GetByIdAsync(cancellationToken, currentTask.ParentTaskId!.Value);

    if (parentTask.AssigneeCode == "system")
    {
        var claimedTasksResponse =
            await camundaService.CamundaClaimedTasks(processData.ProcessInstanceId, parentTask.BlockCode)
            ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);

        if (!Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var stageForCamunda))
            throw new InvalidOperationException($"Unknown process stage: {currentTask.BlockCode}");

        var variables = await BuildCamundaVariablesAsync(
            stageForCamunda, command.Condition, command.Action,
            processData.Id, parentTask.Id, unitOfWork, cancellationToken);

        var submitResponse = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
        {
            TaskId    = claimedTasksResponse.TaskId,
            Variables = variables
        });

        if (!submitResponse.Success)
            throw new HandlerException($"Ошибка при отправке Msg:{submitResponse.Msg}", ErrorCodesEnum.Camunda);

        await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
        await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
        await processTaskService.FinalizeTaskAsync(parentTask, cancellationToken);

        if (!string.IsNullOrWhiteSpace(currentTask.AssigneeCode))
            await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode!, cancellationToken);

        return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };
    }

    await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
    await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
    await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

    await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
    {
        EntityId = parentTask.Id,
        Status   = "Pending"
    }, cancellationToken);

    if (!string.IsNullOrWhiteSpace(parentTask.AssigneeCode) &&
        !string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
    {
        await processTaskService.RefreshUserTaskCacheAsync(parentTask.AssigneeCode, cancellationToken);
    }

    return new BaseResponseDto<SendProcessResponse>
    {
        Data = new SendProcessResponse { Success = true }
    };
}



Добавь внутрь SendProcessCommandHandler.Handle (рядом с другими локальными методами):

async Task HandleEndOfSequentialRoundAsync(
    ProcessTaskEntity parentTask,
    ProcessDataEntity processData,
    ProcessStage stageForCamunda,
    SendProcessCommand command,
    CancellationToken ct)
{
    // если родитель — system, отправляем в Camunda
    if (string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
    {
        var claimedTasksResponse =
            await camundaService.CamundaClaimedTasks(processData.ProcessInstanceId, parentTask.BlockCode)
            ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);

        var variables = await BuildCamundaVariablesAsync(
            stageForCamunda, command.Condition, command.Action,
            processData.Id, parentTask.Id, unitOfWork, ct);

        var submitResponse = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
        {
            TaskId    = claimedTasksResponse.TaskId,
            Variables = variables
        });

        if (!submitResponse.Success)
            throw new HandlerException($"Ошибка при отправке Msg:{submitResponse.Msg}", ErrorCodesEnum.Camunda);

        await processTaskService.FinalizeTaskAsync(parentTask, ct);
    }
    else
    {
        // иначе — поднимаем родителя в Pending
        await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
        {
            EntityId = parentTask.Id,
            Status   = "Pending"
        }, ct);

        if (!string.IsNullOrWhiteSpace(parentTask.AssigneeCode))
            await processTaskService.RefreshUserTaskCacheAsync(parentTask.AssigneeCode, ct);
    }
}
