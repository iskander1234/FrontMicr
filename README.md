public async Task RefreshUserTaskCacheAsync(string userCode, CancellationToken cancellationToken)
{
    var userLower = userCode.ToLower();

    var updatedTasks = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
        cancellationToken,
        p => p.AssigneeCode.ToLower() == userLower && p.Status == "Pending"
    );

    var mapped = mapper.Map<List<GetUserTasksResponse>>(updatedTasks);

    // общий ключ (как у тебя было)
    cache.Set($"tasks:{userLower}", mapped, TimeSpan.FromHours(3));

    // отдельно по стадиям
    foreach (var stageGroup in updatedTasks.GroupBy(t => t.BlockCode))
    {
        var mappedStage = mapper.Map<List<GetUserTasksResponse>>(stageGroup.ToList());
        cache.Set($"tasks:{stageGroup.Key}:{userLower}", mappedStage, TimeSpan.FromHours(3));
    }
}
