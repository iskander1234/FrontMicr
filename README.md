// >>> ДОБАВЬТЕ помощник
async Task UpdateProcessDataBlockToCurrentAsync(Guid processDataId, CancellationToken ct)
{
    // 1) Пытаемся найти активный родительский блок (обычно одна запись на раунд)
    var activeParent = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
        ct,
        t => t.ProcessDataId == processDataId
             && t.ParentTaskId == null
             && t.Status == "Pending"
    );

    // 2) Если нет активного родителя, попробуем взять самый свежий system-переход (опционально)
    if (activeParent == null)
    {
        activeParent = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
            ct,
            t => t.ProcessDataId == processDataId
                 && t.ParentTaskId == null
                 && t.AssigneeCode == "system"
        );
    }

    // 3) Если всё ещё не нашли, выходим (не ломаем поток)
    if (activeParent == null) return;

    await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataBlockChangedEvent
    {
        EntityId = processDataId,
        BlockCode = activeParent.BlockCode,   // <-- не null
        BlockName = activeParent.BlockName    // <-- не null
    }, ct);
}




await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
await processTaskService.FinalizeTaskAsync(parentTask, cancellationToken);


await UpdateProcessDataBlockToCurrentAsync(processData.Id, cancellationToken);



await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
{
    EntityId = parentTask.Id,
    Status = "Pending"
}, ct);

await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataBlockChangedEvent
{
    EntityId = processData.Id,
    BlockCode = parentTask.BlockCode,
    BlockName = parentTask.BlockName
}, ct);
