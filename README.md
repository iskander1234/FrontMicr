// Shared/Queries/Process
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

public sealed class GetUserTasksApprovalQuery       : IRequest<BaseResponseDto<List<GetUserTasksResponse>>> { public string UserCode { get; set; } = default!; }
public sealed class GetUserTasksSigningQuery        : IRequest<BaseResponseDto<List<GetUserTasksResponse>>> { public string UserCode { get; set; } = default!; }
public sealed class GetUserTasksExecutionQuery      : IRequest<BaseResponseDto<List<GetUserTasksResponse>>> { public string UserCode { get; set; } = default!; }
public sealed class GetUserTasksExecutionCheckQuery : IRequest<BaseResponseDto<List<GetUserTasksResponse>>> { public string UserCode { get; set; } = default!; }



using AutoMapper;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using Microsoft.Extensions.Caching.Memory;

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

        // свои Pending по нужному этапу
        var own = await _uow.ProcessTaskRepository.GetByFilterListAsync(
            ct,
            p => p.AssigneeCode != null
                 && p.AssigneeCode.ToLower() == userLower
                 && p.Status == "Pending"
                 && p.BlockCode != null
                 && string.Equals(p.BlockCode, _stageCode, StringComparison.Ordinal));

        var all = new List<ProcessTaskEntity>(own);

        // замещение
        var asDeputy = await _uow.DelegationRepository.GetByFilterListAsync(ct, d => d.DeputyUserCode.ToLower() == userLower);
        if (asDeputy.Any())
        {
            var principals = asDeputy.Select(d => d.PrincipalUserCode.ToLower()).Distinct().ToList();
            var principalTasks = await _uow.ProcessTaskRepository.GetByFilterListAsync(
                ct,
                p => p.AssigneeCode != null
                     && principals.Contains(p.AssigneeCode.ToLower())
                     && p.Status == "Pending"
                     && p.BlockCode != null
                     && string.Equals(p.BlockCode, _stageCode, StringComparison.Ordinal));
            all.AddRange(principalTasks);
        }

        var response = _mapper.Map<List<GetUserTasksResponse>>(all);
        _cache.Set(cacheKey, response, TimeSpan.FromHours(1));
        return new() { Data = response };
    }
}



using AutoMapper;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using Microsoft.Extensions.Caching.Memory;

public sealed class GetUserTasksApprovalQueryHandler
  : GetUserTasksByStageBaseHandler<GetUserTasksApprovalQuery>,
    IRequestHandler<GetUserTasksApprovalQuery, BaseResponseDto<List<GetUserTasksResponse>>>
{
    public GetUserTasksApprovalQueryHandler(IMapper m, IUnitOfWork u, IMemoryCache c) : base(m, u, c, "Approval") { }
}

public sealed class GetUserTasksSigningQueryHandler
  : GetUserTasksByStageBaseHandler<GetUserTasksSigningQuery>,
    IRequestHandler<GetUserTasksSigningQuery, BaseResponseDto<List<GetUserTasksResponse>>>
{
    public GetUserTasksSigningQueryHandler(IMapper m, IUnitOfWork u, IMemoryCache c) : base(m, u, c, "Signing") { }
}

public sealed class GetUserTasksExecutionQueryHandler
  : GetUserTasksByStageBaseHandler<GetUserTasksExecutionQuery>,
    IRequestHandler<GetUserTasksExecutionQuery, BaseResponseDto<List<GetUserTasksResponse>>>
{
    public GetUserTasksExecutionQueryHandler(IMapper m, IUnitOfWork u, IMemoryCache c) : base(m, u, c, "Execution") { }
}

public sealed class GetUserTasksExecutionCheckQueryHandler
  : GetUserTasksByStageBaseHandler<GetUserTasksExecutionCheckQuery>,
    IRequestHandler<GetUserTasksExecutionCheckQuery, BaseResponseDto<List<GetUserTasksResponse>>>
{
    public GetUserTasksExecutionCheckQueryHandler(IMapper m, IUnitOfWork u, IMemoryCache c) : base(m, u, c, "ExecutionCheck") { }
}






[HttpPost("tasks/approval")]
public async Task<IActionResult> GetApproval([FromBody] GetUserTasksQuery q, CancellationToken ct)
    => Ok(await mediator.Send(new GetUserTasksApprovalQuery { UserCode = q.UserCode }, ct));

[HttpPost("tasks/signing")]
public async Task<IActionResult> GetSigning([FromBody] GetUserTasksQuery q, CancellationToken ct)
    => Ok(await mediator.Send(new GetUserTasksSigningQuery { UserCode = q.UserCode }, ct));

[HttpPost("tasks/execution")]
public async Task<IActionResult> GetExecution([FromBody] GetUserTasksQuery q, CancellationToken ct)
    => Ok(await mediator.Send(new GetUserTasksExecutionQuery { UserCode = q.UserCode }, ct));

[HttpPost("tasks/execution-check")]
public async Task<IActionResult> GetExecutionCheck([FromBody] GetUserTasksQuery q, CancellationToken ct)
    => Ok(await mediator.Send(new GetUserTasksExecutionCheckQuery { UserCode = q.UserCode }, ct));

