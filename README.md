public async Task<BaseResponseDto<List<GetUserTasksResponse>>> Handle(
        GetUserTasksQuery query,
        CancellationToken ct)
    {
        string cacheKey = $"tasks:{query.UserCode}";

        if (cache.TryGetValue(cacheKey, out List<GetUserTasksResponse> tasksCache))
            return new BaseResponseDto<List<GetUserTasksResponse>> { Data = tasksCache };

        // 1) получаем всех принципалов, которых я замещаю
        var delegations = await unitOfWork.DelegationRepository
            .GetByDeputyAsync(query.UserCode, ct);

        var principalCodes = delegations
            .Select(d => d.PrincipalUserCode)
            .Distinct()
            .ToList();

        // 2) забираем задачи и для меня, и для принципалов
        var tasks = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
            ct,
            p => (p.AssigneeCode == query.UserCode
                  || principalCodes.Contains(p.AssigneeCode))
                 && p.Status == "Pending"
        );

        var responseList = mapper.Map<List<GetUserTasksResponse>>(tasks);
        cache.Set(cacheKey, responseList, TimeSpan.FromHours(1));

        return new BaseResponseDto<List<GetUserTasksResponse>> { Data = responseList };
    }
