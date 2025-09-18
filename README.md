// 2) Формирование переменных для Camunda
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
            return new Dictionary<string, object> { [varName] = (action == ProcessAction.Submit) };

        case ProcessStage.Signing:
            // sign: Submit => true, Reject => false
            return new Dictionary<string, object> { [varName] = (action == ProcessAction.Submit) };

        case ProcessStage.Registration:
            // registrar: Submit => true, Reject => false (на доработку)
            {
                bool isPositive = action switch
                {
                    ProcessAction.Submit => true,
                    ProcessAction.Reject => false,
                    _ => (condition == ProcessCondition.accept) // запасной вариант
                };
                return new Dictionary<string, object> { [varName] = isPositive };
            }

        case ProcessStage.Approval:
            {
                // agreement: accept => true, remake/reject => false (+учёт вето)
                var isPositive = (condition == ProcessCondition.accept);
                if (isPositive)
                {
                    isPositive = await IsStagePositiveConsideringHistoryAsync(
                        ProcessStage.Approval, isPositive, processDataId, parentTaskId, unitOfWork, ct);
                }
                return new Dictionary<string, object> { [varName] = isPositive };
            }

        case ProcessStage.Execution:
            {
                // executed: accept => true; remake/reject => false; иначе ориентируемся на Action=Submit
                bool isPositive = condition switch
                {
                    ProcessCondition.accept                    => true,
                    ProcessCondition.remake or ProcessCondition.reject => false,
                    _                                          => action == ProcessAction.Submit
                };
                return new Dictionary<string, object> { [varName] = isPositive };
            }

        case ProcessStage.ExecutionCheck:
            {
                // executionCheck: логика как и раньше, только другое имя переменной
                bool isPositive = condition switch
                {
                    ProcessCondition.accept                    => true,
                    ProcessCondition.remake or ProcessCondition.reject => false,
                    _                                          => action == ProcessAction.Submit
                };
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
