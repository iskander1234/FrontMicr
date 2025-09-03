public async Task<BaseResponseDto<List<GetProcessHistoryResponse>>> Handle(GetProcessHistoryQuery query, CancellationToken cancellationToken)
{
    var processTask = await unitOfWork.ProcessTaskRepository
        .GetByFilterAsync(cancellationToken, t => t.Id == query.TaskId)
        ?? throw new HandlerException("Задача не найдена", ErrorCodesEnum.Business);

    var pdId = processTask.ProcessDataId;

    var history = await unitOfWork.ProcessTaskHistoryRepository
        .GetByFilterListAsync(cancellationToken, h => h.ProcessDataId == pdId);

    var historyItems = mapper.Map<List<GetProcessHistoryResponse>>(history);

    var tasks = await unitOfWork.ProcessTaskRepository
        .GetByFilterListAsync(cancellationToken, t => t.ProcessDataId == pdId);

    // Ручной маппинг активных задач в историю
    var taskItems = tasks.Select(t => new GetProcessHistoryResponse
    {
        TaskId       = t.Id,
        ParentTaskId = t.ParentTaskId,
        BlockCode    = t.BlockCode,
        BlockName    = t.BlockName,
        AssigneeCode = t.AssigneeCode,
        AssigneeName = t.AssigneeName,
        Status       = t.Status,
        Comment      = t.Comment,
        Title        = t.Title,
        RegNumber    = t.RegNumber,
        Timestamp    = t.Created   // <-- ключевое: дата для сортировки/дерева
    }).ToList();

    var flat = historyItems
        .Concat(taskItems)
        .OrderBy(x => x.Timestamp)
        .ToList();

    var tree = processService.BuildHistoryTree(flat);

    return new BaseResponseDto<List<GetProcessHistoryResponse>> { Data = tree };
}
