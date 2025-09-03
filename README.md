var tasks = await unitOfWork.ProcessTaskRepository
    .GetByFilterListAsync(
        cancellationToken,
        t => t.ProcessDataId == pdId
             && (t.AssigneeCode == null || t.AssigneeCode.ToLower() != "system")
    );
