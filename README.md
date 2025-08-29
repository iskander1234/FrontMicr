 public async Task RefreshUserTaskCacheAsync(string userCode, CancellationToken cancellationToken)
        {
            var updatedTasks = await unitOfWork.ProcessTaskRepository
                .GetByFilterListAsync(cancellationToken, p => p.AssigneeCode.ToLower() == userCode.ToLower() && p.Status == "Pending");

            var mapped = mapper.Map<List<GetUserTasksResponse>>(updatedTasks);
            cache.Set($"tasks:{userCode}", mapped, TimeSpan.FromHours(3));
        }
