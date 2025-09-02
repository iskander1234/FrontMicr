// ЗАМЕСТИТЕЛЬ
var asDeputy = await _uow.DelegationRepository.GetByFilterListAsync(
    ct, d => d.DeputyUserCode != null && d.DeputyUserCode.ToLower() == userLower);

if (asDeputy.Any())
{
    var principals = asDeputy
        .Where(d => d.PrincipalUserCode != null)
        .Select(d => d.PrincipalUserCode!.ToLower())
        .Distinct()
        .ToList();

    var principalTasks = await _uow.ProcessTaskRepository.GetByFilterListAsync(
        ct,
        p => p.AssigneeCode != null
             && principals.Contains(p.AssigneeCode.ToLower())
             && p.Status == "Pending"
             && p.BlockCode == _stageCode
    );

    all.AddRange(principalTasks);
}

// dedup + сортировка (новые сверху)
var ordered = all
    .GroupBy(t => t.Id)            // убрать дубликаты
    .Select(g => g.First())
    .OrderByDescending(t => t.Created)
    .ToList();

var response = _mapper.Map<List<GetUserTasksResponse>>(ordered);
_cache.Set(cacheKey, response, TimeSpan.FromHours(1));
return new() { Data = response };
