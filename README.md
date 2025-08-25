if (parentTask.AssigneeCode == "system")
{
    var claimedTasksResponse =
        await camundaService.CamundaClaimedTasks(processData.ProcessInstanceId, parentTask.BlockCode)
        ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);

    if (!Enum.TryParse<ProcessStage>(parentTask.BlockCode, out var stage))
        throw new InvalidOperationException($"Unknown process stage: {parentTask.BlockCode}");

    // >>> НОВОЕ: получаем правильные булевы переменные под конкретный этап
    var variables = await BuildCamundaVariablesAsync(
        stage,
        command.Condition,         // accept/remake/reject
        processData.Id,
        parentTask.Id,             // нужен для Approval-раунда
        unitOfWork,
        cancellationToken);

    var submitResponse = await camundaService.CamundaSubmitTask(
        new CamundaSubmitTaskRequest
        {
            TaskId    = claimedTasksResponse.TaskId,
            Variables = variables
        });

    if (!submitResponse.Success)
        throw new HandlerException($"Ошибка при отправке Msg:{submitResponse.Msg}", ErrorCodesEnum.Camunda);

    await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
    await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
    await processTaskService.FinalizeTaskAsync(parentTask, cancellationToken);

    return new BaseResponseDto<SendProcessResponse>
    {
        Data = new SendProcessResponse { Success = true }
    };
}





private static string GetCamundaVarNameForStage(ProcessStage stage) => stage switch
{
    ProcessStage.Approval       => "agreement",
    ProcessStage.Signing        => "sign",
    ProcessStage.Execution      => "executed",        // если в BPMN ждёшь другое имя — поменяй тут
    ProcessStage.ExecutionCheck => "checked",         // при необходимости переименуй
    ProcessStage.Rework         => "agreement",       // обычно на этот этап не сабмитят, но оставим дефолт
    _                           => "agreement"        // безопасный дефолт
};

private static bool IsPositiveByCondition(ProcessCondition condition) =>
    condition == ProcessCondition.accept;

// Для Approval учитываем коллективное «вето»
private static async Task<bool> IsApprovalPositiveConsideringHistoryAsync(
    bool currentIsPositive,
    Guid processDataId,
    Guid? parentTaskId,
    IUnitOfWork unitOfWork,
    CancellationToken ct)
{
    if (!currentIsPositive) return false; // уже отрицательно — дальше не проверяем

    // Ищем в истории по этому же раунду (ParentTaskId) любой remake/reject
    var negativeCount = await unitOfWork.ProcessTaskHistoryRepository.CountAsync(
        ct,
        h => h.ProcessDataId == processDataId
             && h.BlockCode == ProcessStage.Approval.ToString()
             && h.ParentTaskId == parentTaskId
             && (h.Condition == nameof(ProcessCondition.remake)
                 || h.Condition == nameof(ProcessCondition.reject)));

    return negativeCount == 0; // если были отрицательные — итог false
}

private static async Task<Dictionary<string, object>> BuildCamundaVariablesAsync(
    ProcessStage stage,
    ProcessCondition condition,
    Guid processDataId,
    Guid? parentTaskId,
    IUnitOfWork unitOfWork,
    CancellationToken ct)
{
    var varName = GetCamundaVarNameForStage(stage);

    bool isPositive = IsPositiveByCondition(condition);

    if (stage == ProcessStage.Approval)
        isPositive = await IsApprovalPositiveConsideringHistoryAsync(
            isPositive, processDataId, parentTaskId, unitOfWork, ct);

    // На других этапах (Signing/Execution/…) просто маппим accept→true, remake/reject→false
    return new Dictionary<string, object> { [varName] = isPositive };
}



await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(new ProcessTaskHistoryCreatedEvent
{
    // ...
    Condition = command.Condition.ToString(), // <<< важно для подсчёта «вето»
    // ...
}, cancellationToken);
