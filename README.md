var processTask = await unitOfWork.ProcessTaskRepository
    .GetByFilterAsync(cancellationToken, t => t.Id == query.TaskId)
    ?? throw new HandlerException("Задача не найдена", ErrorCodesEnum.Business);

// системный родитель раунда: если есть ParentTaskId – берём его, иначе сам task как корень
var rootId = processTask.ParentTaskId ?? processTask.Id;

// собираем все Id задач в рамках этого «раунда» (включая делегирования любого уровня)
var scopeIds = await CollectRoundScopeAsync(rootId, unitOfWork, cancellationToken);

// история только в пределах scope + только Execution
var execCode = ProcessStage.Execution.ToString();
var history = await unitOfWork.ProcessTaskHistoryRepository
    .GetByFilterListAsync(
        cancellationToken,
        h => h.ProcessDataId == processTask.ProcessDataId
             && h.BlockCode == execCode
             && (
                    (h.ParentTaskId.HasValue && scopeIds.Contains(h.ParentTaskId.Value))
                 || (h.TaskId != Guid.Empty && scopeIds.Contains(h.TaskId))
                )
    );

// дальше как было
var historyItems = mapper.Map<List<GetProcessHistoryResponse>>(history.OrderBy(h => h.Timestamp));
var tree = processService.BuildHistoryTree(historyItems);
return new BaseResponseDto<List<GetExecutionProcessHistoryResponse>>
{
    Data = mapper.Map<List<GetExecutionProcessHistoryResponse>>(tree)
};

// --- helper внутри класса/файла ---
static async Task<HashSet<Guid>> CollectRoundScopeAsync(Guid rootId, IUnitOfWork uow, CancellationToken ct)
{
    var scope = new HashSet<Guid> { rootId };
    var q = new Queue<Guid>();
    q.Enqueue(rootId);

    while (q.Count > 0)
    {
        var pid = q.Dequeue();
        // дети текущего pid
        var children = await uow.ProcessTaskRepository.GetByFilterListAsync(
            ct, t => t.ParentTaskId.HasValue && t.ParentTaskId.Value == pid);

        foreach (var ch in children)
        {
            if (scope.Add(ch.Id))
                q.Enqueue(ch.Id);
        }
    }
    return scope;
}





var execCode = ProcessStage.Execution.ToString();

// находим последний system-parent «Execution» (AssigneeCode == "system")
var lastSystemExec = (await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
        cancellationToken,
        t => t.ProcessDataId == query.ProcessDataId
             && t.BlockCode == execCode
             && t.AssigneeCode == "system"))
    .OrderByDescending(t => t.Created) // или по Timestamp/Id — как у тебя принято
    .FirstOrDefault();

if (lastSystemExec is null)
{
    // fallback: как было раньше (вся история Execution по заявке)
    var historyAll = await unitOfWork.ProcessTaskHistoryRepository
        .GetByFilterListAsync(
            cancellationToken,
            h => h.ProcessDataId == query.ProcessDataId && h.BlockCode == execCode
        );

    var itemsAll = mapper.Map<List<GetProcessHistoryResponse>>(historyAll.OrderBy(h => h.Timestamp));
    var treeAll = processService.BuildHistoryTree(itemsAll);
    return new BaseResponseDto<List<GetExecutionProcessHistoryResponse>>
    {
        Data = mapper.Map<List<GetExecutionProcessHistoryResponse>>(treeAll)
    };
}

// соберём scope последнего раунда
var scopeIds = await CollectRoundScopeAsync(lastSystemExec.Id, unitOfWork, cancellationToken);

// история только по этому раунду
var history = await unitOfWork.ProcessTaskHistoryRepository
    .GetByFilterListAsync(
        cancellationToken,
        h => h.ProcessDataId == query.ProcessDataId
             && h.BlockCode == execCode
             && (
                    (h.ParentTaskId.HasValue && scopeIds.Contains(h.ParentTaskId.Value))
                 || (h.TaskId != Guid.Empty && scopeIds.Contains(h.TaskId))
                )
    );

// дальше как было
var historyItems = mapper.Map<List<GetProcessHistoryResponse>>(history.OrderBy(h => h.Timestamp));
var tree = processService.BuildHistoryTree(historyItems);

return new BaseResponseDto<List<GetExecutionProcessHistoryResponse>>
{
    Data = mapper.Map<List<GetExecutionProcessHistoryResponse>>(tree)
};



