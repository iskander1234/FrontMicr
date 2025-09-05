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
        var cacheKey = $"tasks:{_stageCode}:{userLower}";

        if (_cache.TryGetValue(cacheKey, out List<GetUserTasksResponse> cached))
            return new() { Data = cached };

        // СВОИ Pending по нужному этапу
        var own = await _uow.ProcessTaskRepository.GetByFilterListAsync(
            ct,
            p => p.AssigneeCode != null
                 && p.AssigneeCode.ToLower() == userLower
                 && p.Status == "Pending"
                 && p.BlockCode == _stageCode   // <-- ВАЖНО: без StringComparison
        );

        var all = new List<ProcessTaskEntity>(own);

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

        // сортировка (новые сверху)
        var ordered = all
            .GroupBy(t => t.Id)            // убрать дубликаты
            .Select(g => g.First())
            .OrderByDescending(t => t.Created)
            .ToList();

        var response = _mapper.Map<List<GetUserTasksResponse>>(ordered);
        _cache.Set(cacheKey, response, TimeSpan.FromHours(1));
        return new() { Data = response };
    }
}
