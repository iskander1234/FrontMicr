using AutoMapper;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using Microsoft.Extensions.Caching.Memory;

namespace BpmBaseApi.Application.QueryHandlers.Process;

public abstract class GetUserTasksByStageBaseHandler<TRequest>
    : IRequestHandler<TRequest, BaseResponseDto<List<GetUserTasksResponse>>>
    where TRequest : IRequest<BaseResponseDto<List<GetUserTasksResponse>>>
{
    public const string CachePrefix = "tasks";
    public static readonly string[] StageCodes = new[] { "Approval", "Signing", "Execution", "ExecutionCheck" };

    // универсальный генератор ключей
    public static string BuildCacheKey(string userCode, string? stageCode = null)
    {
        var u = userCode.ToLowerInvariant();
        return stageCode is null
            ? $"{CachePrefix}:{u}"                 // общий список (если где-то используешь)
            : $"{CachePrefix}:{stageCode}:{u}";    // по стадии
    }

    private readonly IMapper _mapper;
    private readonly IUnitOfWork _uow;
    private readonly IMemoryCache _cache;
    private readonly string _stageCode; // "Approval" | "Signing" | "Execution" | "ExecutionCheck"

    protected GetUserTasksByStageBaseHandler(IMapper mapper, IUnitOfWork uow, IMemoryCache cache, string stageCode)
    { _mapper = mapper; _uow = uow; _cache = cache; _stageCode = stageCode; }

    private static string GetUserCode(TRequest req)
        => (string?)typeof(TRequest).GetProperty("UserCode")?.GetValue(req)
           ?? throw new InvalidOperationException("UserCode is required");

    public async Task<BaseResponseDto<List<GetUserTasksResponse>>> Handle(TRequest request, CancellationToken ct)
    {
        var userCode = GetUserCode(request);
        var userLower = userCode.ToLowerInvariant();

        var cacheKey = BuildCacheKey(userLower, _stageCode);  // <<< используем общий генератор

        if (_cache.TryGetValue(cacheKey, out List<GetUserTasksResponse> cached))
            return new() { Data = cached };

        // СВОИ Pending по нужному этапу
        var own = await _uow.ProcessTaskRepository.GetByFilterListAsync(
            ct,
            p => p.AssigneeCode != null
                 && p.AssigneeCode.ToLower() == userLower
                 && p.Status == "Pending"
                 && p.BlockCode == _stageCode   // без StringComparison — EF сможет перевести
        );

        var all = new List<ProcessTaskEntity>(own);

        // ЗАМЕСТИТЕЛЬ
        var asDeputy = await _uow.DelegationRepository.GetByFilterListAsync(
            ct, d => d.DeputyUserCode.ToLower() == userLower);

        if (asDeputy.Any())
        {
            var principals = asDeputy
                .Select(d => d.PrincipalUserCode.ToLower())
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

        var response = _mapper.Map<List<GetUserTasksResponse>>(all);

        _cache.Set(cacheKey, response, TimeSpan.FromHours(1));
        return new() { Data = response };
    }
}


public async Task RefreshUserTaskCacheAsync(string userCode, CancellationToken ct)
{
    var u = userCode.ToLowerInvariant();

    // общий (если где-то используется старый список без стадий)
    _cache.Remove(GetUserTasksByStageBaseHandler<IRequest<BaseResponseDto<List<GetUserTasksResponse>>>>.BuildCacheKey(u, null));

    // по стадиям
    foreach (var stage in GetUserTasksByStageBaseHandler<IRequest<BaseResponseDto<List<GetUserTasksResponse>>>>.StageCodes)
    {
        _cache.Remove(GetUserTasksByStageBaseHandler<IRequest<BaseResponseDto<List<GetUserTasksResponse>>>>.BuildCacheKey(u, stage));
    }

    await Task.CompletedTask;
}






await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
try
{
    await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);
}
catch { /* best effort */ }






