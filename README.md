public async Task RefreshUserTaskCacheAsync(string userCode, CancellationToken cancellationToken)
{
    if (string.IsNullOrWhiteSpace(userCode)) return;

    var userNorm = NormalizeUser(userCode);

    // 1) Снести прошлые ключи
    cache.Remove(UserRootKey(userNorm));
    if (cache.TryGetValue<string[]>(UserStagesIndexKey(userNorm), out var oldStageKeys) && oldStageKeys is not null)
    {
        foreach (var k in oldStageKeys) cache.Remove(k);
        cache.Remove(UserStagesIndexKey(userNorm));
    }

    // 2) Актуальные Pending пользователя (только свои)
    var updatedTasks = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
        cancellationToken,
        p => p.AssigneeCode != null
             && p.Status == "Pending"
             && p.AssigneeCode.ToLower() == userNorm
    );

    // ⚠️ ВАЖНО: корневой ключ — отсортированно (если он где-то используется)
    var orderedAll = updatedTasks
        .OrderByDescending(t => t.Created)
        .ThenBy(t => t.Id) // стабильность при одинаковых Created
        .ToList();

    var mappedAll = mapper.Map<List<GetUserTasksResponse>>(orderedAll);

    var ttl = TimeSpan.FromMinutes(1);
    cache.Set(UserRootKey(userNorm), mappedAll, ttl);

    // 3) НЕ пишем tasks:{Stage}:{user} здесь, только инвалидация выше!
    //    Пусть GetUserTasksByStageBaseHandler построит и запишет их сам (со своими + замещением + сортировкой).
    //    => Не сохраняем новый индекс стадий.
}
