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
        {
            // registrar: Submit => true, Reject => false (на доработку)
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
            // 1) Пытаемся определить исход по Condition
            bool? isPositiveMaybe = condition switch
            {
                ProcessCondition.accept => true,
                ProcessCondition.remake or ProcessCondition.reject => false,
                _ => null
            };

            // 2) Если Condition нет — fallback на Action
            if (isPositiveMaybe is null)
            {
                isPositiveMaybe = action switch
                {
                    ProcessAction.Submit => true,   // фронт часто шлёт только Submit
                    ProcessAction.Reject => false,
                    ProcessAction.Cancel => false,
                    _ => (bool?)null
                };
            }

            // 3) Если всё ещё null — считаем false (или бросай HandlerException по желанию)
            bool isPositive = isPositiveMaybe ?? false;

            // 4) «Вето» в рамках того же раунда (родителя)
            if (isPositive)
            {
                isPositive = await IsStagePositiveConsideringHistoryAsync(
                    ProcessStage.Approval, isPositive, processDataId, parentTaskId, unitOfWork, ct);
            }

            return new Dictionary<string, object> { [varName] = isPositive };
        }

        case ProcessStage.Execution:
        {
            // executed: accept => true, remake/reject => false, иначе Submit => true
            bool isPositive = condition switch
            {
                ProcessCondition.accept => true,
                ProcessCondition.remake or ProcessCondition.reject => false,
                _ => action == ProcessAction.Submit
            };
            return new Dictionary<string, object> { [varName] = isPositive };
        }

        case ProcessStage.ExecutionCheck:
        {
            // executionCheck: та же логика, только другое имя переменной
            bool isPositive = condition switch
            {
                ProcessCondition.accept => true,
                ProcessCondition.remake or ProcessCondition.reject => false,
                _ => action == ProcessAction.Submit
            };
            return new Dictionary<string, object> { [varName] = isPositive };
        }

        default:
        {
            // безопасный дефолт
            var isPositive = (condition == ProcessCondition.accept);
            return new Dictionary<string, object> { [varName] = isPositive };
        }
    }
}
