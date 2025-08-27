if (!Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var stage))
    throw new InvalidOperationException($"Unknown process stage: {currentTask.BlockCode}");



bool isNegative = stage switch
{
    ProcessStage.Rework   => command.Action == ProcessAction.Cancel,
    ProcessStage.Signing  => command.Action == ProcessAction.Reject,
    _ /* Approval и прочее */ => command.Condition is ProcessCondition.remake or ProcessCondition.reject
};




var pendingSiblings = await unitOfWork.ProcessTaskRepository.CountAsync(
    cancellationToken,
    t => t.ParentTaskId == currentTask.ParentTaskId && t.Id != currentTask.Id /*&& t.Status == "Pending"*/);



private static async Task<Dictionary<string, object>> BuildCamundaVariablesAsync(
    ProcessStage stage,
    ProcessCondition condition,
    ProcessAction action,
    Guid processDataId,
    Guid? parentTaskId,
    IUnitOfWork unitOfWork,
    CancellationToken ct)
{
    var varName = GetCamundaVarNameForStage(stage);

    switch (stage)
    {
        case ProcessStage.Rework:
            // refinement: Submit => true, Cancel => false
            return new Dictionary<string, object>
            {
                [varName] = (action == ProcessAction.Submit)
            };

        case ProcessStage.Signing:
            // sign: Submit => true, Reject => false
            return new Dictionary<string, object>
            {
                [varName] = (action == ProcessAction.Submit)
            };

        case ProcessStage.Approval:
        {
            // agreement: accept => true, remake/reject => false
            var isPositive = (condition == ProcessCondition.accept);

            if (isPositive)
            {
                // «вето» в рамках того же родителя (раунда)
                isPositive = await IsStagePositiveConsideringHistoryAsync(
                    ProcessStage.Approval, isPositive, processDataId, parentTaskId, unitOfWork, ct);
            }

            return new Dictionary<string, object> { [varName] = isPositive };
        }

        default:
        {
            // дефолт: accept => true
            var isPositive = (condition == ProcessCondition.accept);
            return new Dictionary<string, object> { [varName] = isPositive };
        }
    }
}



