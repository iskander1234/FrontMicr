private static async Task<bool> IsStagePositiveConsideringHistoryAsync(
    ProcessStage stage,
    bool currentIsPositive,
    Guid processDataId,
    Guid? parentTaskId,                 // сюда ты и так передаёшь system-родителя
    IUnitOfWork unitOfWork,
    CancellationToken ct)
{
    if (!currentIsPositive) return false;
    if (parentTaskId is null) return currentIsPositive;

    var stageCode = stage.ToString();

    // 1) Все "дети раунда": задачи, у которых ParentTaskId == system-родителю
    var roundLevel1 = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
        ct,
        t => t.ParentTaskId == parentTaskId
    );
    var l1Ids = roundLevel1.Select(t => t.Id).ToList();

    // 2) Делегированные задачи (1 уровень вниз): ParentTaskId ∈ Level1
    List<Guid> l2Ids = new();
    if (l1Ids.Count > 0)
    {
        var roundLevel2 = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
            ct,
            t => t.ParentTaskId != null && l1Ids.Contains(t.ParentTaskId.Value)
        );
        l2Ids = roundLevel2.Select(t => t.Id).ToList();
    }

    // 3) Собираем множество id, относящихся к текущему "раунду" (system-родитель + дети + делегаты)
    var scopeIds = new HashSet<Guid>(l1Ids.Concat(l2Ids)) { parentTaskId.Value };

    // 4) Был ли где-то в этом раунде remake/reject?
    var negativeCount = await unitOfWork.ProcessTaskHistoryRepository.CountAsync(
        ct,
        h => h.ProcessDataId == processDataId
             && h.BlockCode == stageCode
             && (
                    // История, привязанная к родителю/детям раунда
                    (h.ParentTaskId != null && scopeIds.Contains(h.ParentTaskId.Value))
                 || (h.TaskId != null && scopeIds.Contains(h.TaskId.Value))
                )
             && (h.Condition == nameof(ProcessCondition.remake)
                 || h.Condition == nameof(ProcessCondition.reject))
    );

    // если находили remake/reject в этом раунде — итог НЕ положительный
    return negativeCount == 0;
}
