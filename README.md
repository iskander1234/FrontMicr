var variables = await BuildCamundaVariablesAsync(
    stage,
    command.Condition,      // accept/remake/reject
    command.Action,         // <<< ДОБАВИЛИ
    processData.Id,
    parentTask.Id,          // раунд по ParentTaskId
    unitOfWork,
    cancellationToken);




private static string GetCamundaVarNameForStage(ProcessStage stage) => stage switch
{
    ProcessStage.Approval       => "agreement",
    ProcessStage.Signing        => "sign",
    ProcessStage.Execution      => "executed",
    ProcessStage.ExecutionCheck => "checked",
    ProcessStage.Rework         => "refinement",   // <<< БЫЛО "agreement", теперь "refinement"
    _                           => "agreement"
};





private static async Task<bool> IsStagePositiveConsideringHistoryAsync(
    ProcessStage stage,
    bool currentIsPositive,
    Guid processDataId,
    Guid? parentTaskId,
    IUnitOfWork unitOfWork,
    CancellationToken ct)
{
    if (!currentIsPositive) return false;

    var stageCode = stage.ToString();

    var negativeCount = await unitOfWork.ProcessTaskHistoryRepository.CountAsync(
        ct,
        h => h.ProcessDataId == processDataId
             && h.BlockCode == stageCode
             && h.ParentTaskId == parentTaskId
             && (h.Condition == nameof(ProcessCondition.remake)
                 || h.Condition == nameof(ProcessCondition.reject)));

    return negativeCount == 0;
}




private static async Task<Dictionary<string, object>> BuildCamundaVariablesAsync(
    ProcessStage stage,
    ProcessCondition condition,
    ProcessAction action,          // <<< ДОБАВИЛИ
    Guid processDataId,
    Guid? parentTaskId,
    IUnitOfWork unitOfWork,
    CancellationToken ct)
{
    var varName = GetCamundaVarNameForStage(stage);

    // Базовое булево: accept -> true, remake/reject -> false
    bool isPositive = IsPositiveByCondition(condition);

    if (stage == ProcessStage.Rework)
    {
        if (action == ProcessAction.Cancel)
        {
            // Отмена на доработке — жестко false и Camunda должна по диаграмме закрыть процесс
            return new Dictionary<string, object> { [varName] = false };
        }

        // Submit на Rework — снова проверяем согласующих (вето)
        isPositive = await IsStagePositiveConsideringHistoryAsync(
            ProcessStage.Rework, isPositive, processDataId, parentTaskId, unitOfWork, ct);

        return new Dictionary<string, object> { [varName] = isPositive };
    }

    // Approval — как было: учитываем «вето»
    if (stage == ProcessStage.Approval)
    {
        isPositive = await IsStagePositiveConsideringHistoryAsync(
            ProcessStage.Approval, isPositive, processDataId, parentTaskId, unitOfWork, ct);

        return new Dictionary<string, object> { [varName] = isPositive };
    }

    // Остальные стадии — просто маппинг accept/remake/reject -> true/false
    return new Dictionary<string, object> { [varName] = isPositive };
}




await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(new ProcessTaskHistoryCreatedEvent
{
    // ...
    BlockCode  = currentTask.BlockCode,             // например "Rework" или "Approval"
    ParentTaskId = currentTask.ParentTaskId,
    Condition = command.Condition.ToString(),       // "accept"/"remake"/"reject"
    // ...
}, cancellationToken);



