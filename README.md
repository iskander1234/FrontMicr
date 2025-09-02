// ... после all.AddRange(principalTasks);

var ordered = all
    .OrderByDescending(t => t.Created)   // новые сверху
    .ToList();

var response = _mapper.Map<List<GetUserTasksResponse>>(ordered);
_cache.Set(cacheKey, response, TimeSpan.FromHours(1));
return new() { Data = response };
