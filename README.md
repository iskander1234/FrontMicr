public static class UserTasksCache
{
    public const string Prefix = "tasks";
    public static readonly string[] Stages = { "Approval", "Signing", "Execution", "ExecutionCheck" };

    public static string Build(string userCode, string stageCode)
        => $"{Prefix}:{stageCode}:{userCode?.ToLowerInvariant()}";

    public static void InvalidateStage(IMemoryCache cache, string? userCode, string stageCode)
    {
        if (string.IsNullOrWhiteSpace(userCode)) return;
        cache.Remove(Build(userCode, stageCode));
    }

    public static void InvalidateAll(IMemoryCache cache, string? userCode)
    {
        if (string.IsNullOrWhiteSpace(userCode)) return;
        foreach (var s in Stages) cache.Remove(Build(userCode, s));
        cache.Remove($"{Prefix}:{userCode.ToLowerInvariant()}"); // если где-то есть общий ключ
    }
}



GetUserTasksByStageBaseHandler

var cacheKey = UserTasksCache.Build(userLower, _stageCode);
if (_cache.TryGetValue(cacheKey, out List<GetUserTasksResponse> cached))
    return new() { Data = cached };
// ... после маппинга:
_cache.Set(cacheKey, response, TimeSpan.FromHours(1));








var stageCode = currentTask.BlockCode;
var oldAssignee = currentTask.AssigneeCode;
var newAssignee = recipients.FirstOrDefault()?.UserCode;

// Делегирование
if (command.Action == ProcessAction.Delegate && recipients.Any())
{
    await processTaskService.HandleDelegationAsync(currentTask, processData, command, recipients, cancellationToken);

    // инвалидация кешей (старый и новый исполнитель по этому этапу)
    UserTasksCache.InvalidateStage(cache, oldAssignee, stageCode);
    UserTasksCache.InvalidateStage(cache, newAssignee, stageCode);

    return new BaseResponseDto<SendProcessResponse>
    {
        Data = new SendProcessResponse { Success = true }
    };
}




await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
{
    EntityId = next.Id,
    Status = "Pending"
}, cancellationToken);

// инвалидации кешей исполнителей по этому этапу
UserTasksCache.InvalidateStage(cache, currentTask.AssigneeCode, stage.ToString());
UserTasksCache.InvalidateStage(cache, next.AssigneeCode,    stage.ToString());

return new BaseResponseDto<SendProcessResponse>
{
    Data = new SendProcessResponse { Success = true }
};




foreach (var w in siblings.Where(t => t.Status == "Waiting"))
{
    await processTaskService.FinalizeTaskAsync(w, cancellationToken);
    UserTasksCache.InvalidateStage(cache, w.AssigneeCode, stage.ToString());
}

// и для текущего
UserTasksCache.InvalidateStage(cache, currentTask.AssigneeCode, stage.ToString());


UserTasksCache.InvalidateStage(cache, currentTask.AssigneeCode, currentTask.BlockCode);




await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
UserTasksCache.InvalidateStage(cache, currentTask.AssigneeCode, currentTask.BlockCode);

await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
{
    EntityId = parentTask.Id,
    Status = "Pending"
}, cancellationToken);

// если родитель не system и есть исполнитель — можно почистить и его этап (обычно другой блок)
UserTasksCache.InvalidateAll(cache, parentTask.AssigneeCode);
