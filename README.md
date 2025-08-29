 public async Task FinalizeTaskAsync(ProcessTaskEntity task, CancellationToken cancellationToken)
        {
            await unitOfWork.ProcessTaskRepository.DeleteByIdAsync(task.Id, cancellationToken);
            await RefreshUserTaskCacheAsync(task.AssigneeCode, cancellationToken);
        }

 public async Task RefreshUserTaskCacheAsync(string userCode, CancellationToken cancellationToken)
        {
            var updatedTasks = await unitOfWork.ProcessTaskRepository
                .GetByFilterListAsync(cancellationToken, p => p.AssigneeCode.ToLower() == userCode.ToLower() && p.Status == "Pending");

            var mapped = mapper.Map<List<GetUserTasksResponse>>(updatedTasks);
            cache.Set($"tasks:{userCode}", mapped, TimeSpan.FromHours(3));
        }
        
