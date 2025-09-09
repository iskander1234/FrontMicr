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

    // 1) Дочерние задачи "раунда" (дети system-родителя)
    var roundLevel1 = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
        ct,
        t => t.ParentTaskId == parentTaskId
    );
    var l1Ids = roundLevel1.Select(t => t.Id).ToList();

    // 2) Делегированные задачи (1 уровень вниз от Level1)
    var l2Ids = new List<Guid>();
    if (l1Ids.Count > 0)
    {
        var roundLevel2 = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
            ct,
            t => t.ParentTaskId != null && l1Ids.Contains(t.ParentTaskId.Value)
        );
        l2Ids = roundLevel2.Select(t => t.Id).ToList();
    }

    // 3) Собираем scope текущего раунда
    var scopeIds = l1Ids
        .Concat(l2Ids)
        .Append(parentTaskId.Value)
        .Distinct()
        .ToArray(); // массив лучше транслируется в SQL IN

    // 4) Есть ли remake/reject в этом раунде?
    var negativeCount = await unitOfWork.ProcessTaskHistoryRepository.CountAsync(
        ct,
        h => h.ProcessDataId == processDataId
             && h.BlockCode == stageCode
             && (
                    // История, привязанная к родителю/детям раунда
                    (h.ParentTaskId.HasValue && scopeIds.Contains(h.ParentTaskId.Value))
                    // И/или к конкретным задачам (в т.ч. делегированным). TaskId — НЕ nullable Guid.
                 || (h.TaskId != Guid.Empty && scopeIds.Contains(h.TaskId))
                )
             && (h.Condition == nameof(ProcessCondition.remake)
                 || h.Condition == nameof(ProcessCondition.reject))
    );

    return negativeCount == 0;
}
