- var pendingSiblings = await unitOfWork.ProcessTaskRepository.CountAsync(
-     cancellationToken,
-     t => t.ParentTaskId == currentTask.ParentTaskId && t.Id != currentTask.Id /*&& t.Status == "Pending"*/);
+ var pendingSiblings = await unitOfWork.ProcessTaskRepository.CountAsync(
+     cancellationToken,
+     t => t.ParentTaskId == currentTask.ParentTaskId
+          && t.Id != currentTask.Id
+          && t.Status == "Pending");

... // перед Camunda

- if (!Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var stage))
-     throw new InvalidOperationException($"Unknown process stage: {currentTask.BlockCode}");
+ if (!Enum.TryParse<ProcessStage>(parentTask.BlockCode, out var stageForCamunda))
+     throw new InvalidOperationException($"Unknown process stage: {parentTask.BlockCode}");

- var variables = await BuildCamundaVariablesAsync(
-     stage,
+ var variables = await BuildCamundaVariablesAsync(
+     stageForCamunda,
      command.Condition,
      command.Action,
      processData.Id,
      parentTask.Id,
      unitOfWork,
      cancellationToken);
