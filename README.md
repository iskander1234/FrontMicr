public async Task RefreshUserTaskCacheAsync(string? userCode, CancellationToken ct)
{
    if (string.IsNullOrWhiteSpace(userCode)) return;

    var userKey = Normalize(userCode);

    // 1) Снести старые ключи
    cache.Remove($"tasks:{userKey}");
    var stagesIndexKey = $"tasks:{userKey}:__stages";
    if (cache.TryGetValue<string[]>(stagesIndexKey, out var oldStageKeys) && oldStageKeys is not null)
    {
        foreach (var k in oldStageKeys) cache.Remove(k);
        cache.Remove(stagesIndexKey);
    }

    // 2) Прочитать актуально из БД (желательно хранить AssigneeCode уже нормализованным)
    var updatedTasks = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
        ct, p => p.AssigneeCode == userKey && p.Status == "Pending");

    var mapped = mapper.Map<List<GetUserTasksResponse>>(updatedTasks);

    var ttl = TimeSpan.FromSeconds(45); // короткий TTL

    cache.Set($"tasks:{userKey}", mapped, ttl);

    // 3) Пересоздать ключи по стадиям и индекс стадий
    var newStageKeys = new List<string>();
    foreach (var grp in updatedTasks.GroupBy(t => t.BlockCode))
    {
        var stageKey = $"tasks:{grp.Key}:{userKey}";
        var mappedStage = mapper.Map<List<GetUserTasksResponse>>(grp.ToList());
        cache.Set(stageKey, mappedStage, ttl);
        newStageKeys.Add(stageKey);
    }
    cache.Set(stagesIndexKey, newStageKeys.ToArray(), ttl);
}
