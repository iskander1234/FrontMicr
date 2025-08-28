var parentTask = await unitOfWork.ProcessTaskRepository.GetByIdAsync(cancellationToken, currentTask.ParentTaskId.Value);

if (parentTask.AssigneeCode == "system")
{
    var claimedTasksResponse =
        await camundaService.CamundaClaimedTasks(processData.ProcessInstanceId, parentTask.BlockCode)
        ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);

    if (!Enum.TryParse<ProcessStage>(parentTask.BlockCode, out var stageForCamunda))
        throw new InvalidOperationException($"Unknown process stage: {parentTask.BlockCode}");

    var variables = await BuildCamundaVariablesAsync(
        stageForCamunda,
        command.Condition,
        command.Action,
        processData.Id,
        parentTask.Id,
        unitOfWork,
        cancellationToken);

    var submitResponse = await camundaService.CamundaSubmitTask(
        new CamundaSubmitTaskRequest { TaskId = claimedTasksResponse.TaskId, Variables = variables });

    if (!submitResponse.Success)
        throw new HandlerException($"Ошибка при отправке Msg:{submitResponse.Msg}", ErrorCodesEnum.Camunda);

    if (!alreadyLoggedAndFinalized)
    {
        await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
        await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
    }
    await processTaskService.FinalizeTaskAsync(parentTask, cancellationToken);

    return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };
}
