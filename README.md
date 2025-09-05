 public async Task FinalizeTaskAsync(ProcessTaskEntity task, CancellationToken cancellationToken)
        {
            await unitOfWork.ProcessTaskRepository.DeleteByIdAsync(task.Id, cancellationToken);

            // безопасно: null / пробелы игнорим
            if (!string.IsNullOrWhiteSpace(task.AssigneeCode))
                await RefreshUserTaskCacheAsync(task.AssigneeCode!, cancellationToken);
        }

        private static string NormalizeUser(string userCode)
            => userCode.Trim().ToLowerInvariant();

        private static string UserRootKey(string userNorm)
            => $"tasks:{userNorm}";

        private static string UserStagesIndexKey(string userNorm)
            => $"tasks:{userNorm}:__stages";

        private static string StageKey(string stageCode, string userNorm)
            => $"tasks:{stageCode}:{userNorm}";

        public async Task RefreshUserTaskCacheAsync(string userCode, CancellationToken cancellationToken)
        {
            if (string.IsNullOrWhiteSpace(userCode)) return;

            var userNorm = NormalizeUser(userCode);

            // 1) Снести прошлые ключи, чтобы не висели «пустые стадии»
            cache.Remove(UserRootKey(userNorm));
            if (cache.TryGetValue<string[]>(UserStagesIndexKey(userNorm), out var oldStageKeys) &&
                oldStageKeys is not null)
            {
                foreach (var k in oldStageKeys) cache.Remove(k);
                cache.Remove(UserStagesIndexKey(userNorm));
            }

            // 2) Прочитать актуальное Pending (защита от null в БД + case-insensitive)
            var updatedTasks = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
                cancellationToken,
                p => p.AssigneeCode != null
                     && p.Status == "Pending"
                     && p.AssigneeCode.ToLower() == userNorm
            );

            // корневой ключ — отсортированно (если он где-то используется)
            var orderedAll = updatedTasks
                .OrderByDescending(t => t.Created)
                .ThenBy(t => t.Id) // стабильность при одинаковых Created
                .ToList();
            
            var mappedAll = mapper.Map<List<GetUserTasksResponse>>(orderedAll);

            // TTL лучше короткий, но чтобы «не ломать» — вынесем в константу (можешь поставить minutes:1)
            var ttl = TimeSpan.FromMinutes(1);

            // 3) Положить общий ключ
            cache.Set(UserRootKey(userNorm), mappedAll, ttl);

            // 4) Разложить по стадиям + сохранить индекс стадий, чтобы в следующий раз удалить корректно
            var newStageKeys = new List<string>();
            foreach (var grp in updatedTasks.GroupBy(t => t.BlockCode))
            {
                var stageKey = StageKey(grp.Key, userNorm);
                var mappedStage = mapper.Map<List<GetUserTasksResponse>>(grp.ToList());
                cache.Set(stageKey, mappedStage, ttl);
                newStageKeys.Add(stageKey);
            }

            cache.Set(UserStagesIndexKey(userNorm), newStageKeys.ToArray(), ttl);
        }
        
