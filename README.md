private static async Task<Dictionary<string, object>> BuildCamundaVariablesAsync(
    ProcessStage stage,
    ProcessCondition condition,
    ProcessAction action,
    Guid processDataId,
    Guid? parentTaskId,
    IUnitOfWork unitOfWork,
    CancellationToken ct)
{
    // мапперы
    static bool? FromCondition(ProcessCondition c) => c switch
    {
        ProcessCondition.accept                 => true,
        ProcessCondition.remake or ProcessCondition.reject => false,
        _                                       => (bool?)null
    };

    static bool? FromAction(ProcessAction a) => a switch
    {
        ProcessAction.Submit => true,
        ProcessAction.Reject => false,
        ProcessAction.Cancel => false,
        _                    => (bool?)null
    };

    // 1) Execution — в BPMN нет переменной, узел всегда «true»: НИЧЕГО не отправляем.
    if (stage == ProcessStage.Execution)
        return new Dictionary<string, object>(); // пусто => Camunda пойдёт дальше без условия

    var varName = GetCamundaVarNameForStage(stage);

    // 2) Стадии, где приоритет — Condition (с фоллбеком на Action)
    bool conditionDriven = stage is
        ProcessStage.Approval
        or ProcessStage.Signing
        or ProcessStage.ExecutionCheck
        or ProcessStage.Rework
        or ProcessStage.Registration;

    bool isPositive;

    if (conditionDriven)
    {
        var byCond = FromCondition(condition);
        var byAct  = FromAction(action);
        isPositive = byCond ?? byAct ?? false; // если ничего не пришло — false по-умолчанию

        // особый случай «вето» только для Approval
        if (stage == ProcessStage.Approval && isPositive)
        {
            isPositive = await IsStagePositiveConsideringHistoryAsync(
                ProcessStage.Approval, isPositive, processDataId, parentTaskId, unitOfWork, ct);
        }
    }
    else
    {
        // 3) Остальные стадии — только по Action
        isPositive = FromAction(action) ?? false;
    }

    return new Dictionary<string, object> { [varName] = isPositive };
}
