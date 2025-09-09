private static async Task<bool> IsStagePositiveConsideringHistoryAsync(
    ProcessStage stage,
    bool currentIsPositive,
    Guid processDataId,
    Guid? parentTaskId,
    IUnitOfWork unitOfWork,
    CancellationToken ct)
{
    if (!currentIsPositive) return false;
    if (parentTaskId is null) return currentIsPositive;

    var stageCode = stage.ToString();

    // 1) system-родитель текущего раунда
    var root = await unitOfWork.ProcessTaskRepository.GetByIdAsync(ct, parentTaskId.Value)
               ?? throw new HandlerException("Системная задача раунда не найдена", ErrorCodesEnum.Business);

    // 2) Собираем ВСЕ задачи раунда: дети любого уровня от system-родителя (BFS)
    var scope = new HashSet<Guid> { parentTaskId.Value };
    var frontier = new Queue<Guid>();
    frontier.Enqueue(parentTaskId.Value);

    while (frontier.Count > 0)
    {
        var pid = frontier.Dequeue();

        // берём прямых детей pid
        var children = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
            ct,
            t => t.ParentTaskId != null && t.ParentTaskId == pid
        );

        foreach (var ch in children)
        {
            if (scope.Add(ch.Id))
                frontier.Enqueue(ch.Id);
        }
    }

    // 3) Проверяем, есть ли remake/reject в рамках этого «поддерева» (TaskId / ParentTaskId в scope)
    //    + запасной вариант: история без ParentTaskId, но с BlockCode == stageCode и Timestamp >= созданию root
    var negativeCount = await unitOfWork.ProcessTaskHistoryRepository.CountAsync(
        ct,
        h => h.ProcessDataId == processDataId
             && h.BlockCode == stageCode
             && (
                    (h.ParentTaskId.HasValue && scope.Contains(h.ParentTaskId.Value))
                 || (h.TaskId != Guid.Empty && scope.Contains(h.TaskId))
                 || (!h.ParentTaskId.HasValue && h.TaskId == Guid.Empty && h.Timestamp >= root.Created)
                )
             && (h.Condition != null && (
                    h.Condition.Equals(nameof(ProcessCondition.remake), StringComparison.OrdinalIgnoreCase)
                 || h.Condition.Equals(nameof(ProcessCondition.reject), StringComparison.OrdinalIgnoreCase)))
    );

    return negativeCount == 0;
}
