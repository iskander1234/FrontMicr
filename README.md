// текущего — убрать из кэша его Pending (важно для positive пути)
await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);
