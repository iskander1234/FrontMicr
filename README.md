// =============================
// Queries (по одному на этап)
// =============================
using MediatR;
using BpmBaseApi.Shared.Responses.Process;
using BpmBaseApi.Shared.Dtos;

namespace BpmBaseApi.Shared.Queries.Process
{
    public class GetUserTasksApprovalQuery : IRequest<BaseResponseDto<List<GetUserTasksResponse>>>
    {
        public string UserCode { get; set; } = default!;
    }

    public class GetUserTasksSigningQuery : IRequest<BaseResponseDto<List<GetUserTasksResponse>>>
    {
        public string UserCode { get; set; } = default!;
    }

    public class GetUserTasksExecutionQuery : IRequest<BaseResponseDto<List<GetUserTasksResponse>>>
    {
        public string UserCode { get; set; } = default!;
    }

    public class GetUserTasksExecutionCheckQuery : IRequest<BaseResponseDto<List<GetUserTasksResponse>>>
    {
        public string UserCode { get; set; } = default!;
    }
}


// ================================================
// Базовая реализация для переиспользования логики
// ================================================
using AutoMapper;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using Microsoft.Extensions.Caching.Memory;

namespace BpmBaseApi.Application.QueryHandlers.Process
{
    public abstract class GetUserTasksByStageBaseHandler<TRequest> : IRequestHandler<TRequest, BaseResponseDto<List<GetUserTasksResponse>>>
        where TRequest : IRequest<BaseResponseDto<List<GetUserTasksResponse>>>
    {
        private readonly IMapper _mapper;
        private readonly IUnitOfWork _unitOfWork;
        private readonly IMemoryCache _cache;
        private readonly string _stageCode; // "Approval" | "Signing" | "Execution" | "ExecutionCheck"

        protected GetUserTasksByStageBaseHandler(
            IMapper mapper,
            IUnitOfWork unitOfWork,
            IMemoryCache cache,
            string stageCode)
        {
            _mapper = mapper;
            _unitOfWork = unitOfWork;
            _cache = cache;
            _stageCode = stageCode;
        }

        // Достаём UserCode через рефлексию, чтобы не дублировать код
        private static string GetUserCodeFromRequest(TRequest request)
        {
            var prop = typeof(TRequest).GetProperty("UserCode");
            if (prop == null)
                throw new InvalidOperationException($"{typeof(TRequest).Name} должен иметь свойство UserCode");
            return (string)(prop.GetValue(request) ?? throw new InvalidOperationException("UserCode is null"));
        }

        public async Task<BaseResponseDto<List<GetUserTasksResponse>>> Handle(
            TRequest request,
            CancellationToken cancellationToken)
        {
            var userCode = GetUserCodeFromRequest(request);
            var userCodeLower = userCode.ToLowerInvariant();
            var cacheKey = $"tasks:{_stageCode}:{userCodeLower}";

            if (_cache.TryGetValue(cacheKey, out List<GetUserTasksResponse> cached))
                return new BaseResponseDto<List<GetUserTasksResponse>> { Data = cached };

            // 1) Свои Pending задачи по конкретному этапу
            var ownTasks = await _unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
                cancellationToken,
                p => p.AssigneeCode.ToLower() == userCodeLower
                     && p.Status == "Pending"
                     && p.BlockCode == _stageCode);

            var all = new List<ProcessTaskEntity>(ownTasks);

            // 2) Я — заместитель?
            var asDeputy = await _unitOfWork.DelegationRepository.GetByFilterListAsync(
                cancellationToken,
                d => d.DeputyUserCode.ToLower() == userCodeLower);

            if (asDeputy.Any())
            {
                var principalCodes = asDeputy
                    .Select(d => d.PrincipalUserCode.ToLowerInvariant())
                    .Distinct()
                    .ToList();

                var principalTasks = await _unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
                    cancellationToken,
                    p => principalCodes.Contains(p.AssigneeCode.ToLower())
                         && p.Status == "Pending"
                         && p.BlockCode == _stageCode);

                all.AddRange(principalTasks);
            }

            var response = _mapper.Map<List<GetUserTasksResponse>>(all);
            _cache.Set(cacheKey, response, TimeSpan.FromHours(1));

            return new BaseResponseDto<List<GetUserTasksResponse>> { Data = response };
        }
    }
}


// =====================================================
// Конкретные хендлеры на каждый этап (4 штуки)
// =====================================================
using AutoMapper;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Queries.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using Microsoft.Extensions.Caching.Memory;

namespace BpmBaseApi.Application.QueryHandlers.Process
{
    public class GetUserTasksApprovalQueryHandler
        : GetUserTasksByStageBaseHandler<GetUserTasksApprovalQuery>,
          IRequestHandler<GetUserTasksApprovalQuery, BaseResponseDto<List<GetUserTasksResponse>>>
    {
        public GetUserTasksApprovalQueryHandler(IMapper mapper, IUnitOfWork unitOfWork, IMemoryCache cache)
            : base(mapper, unitOfWork, cache, "Approval") { }
    }

    public class GetUserTasksSigningQueryHandler
        : GetUserTasksByStageBaseHandler<GetUserTasksSigningQuery>,
          IRequestHandler<GetUserTasksSigningQuery, BaseResponseDto<List<GetUserTasksResponse>>>
    {
        public GetUserTasksSigningQueryHandler(IMapper mapper, IUnitOfWork unitOfWork, IMemoryCache cache)
            : base(mapper, unitOfWork, cache, "Signing") { }
    }

    public class GetUserTasksExecutionQueryHandler
        : GetUserTasksByStageBaseHandler<GetUserTasksExecutionQuery>,
          IRequestHandler<GetUserTasksExecutionQuery, BaseResponseDto<List<GetUserTasksResponse>>>
    {
        public GetUserTasksExecutionQueryHandler(IMapper mapper, IUnitOfWork unitOfWork, IMemoryCache cache)
            : base(mapper, unitOfWork, cache, "Execution") { }
    }

    public class GetUserTasksExecutionCheckQueryHandler
        : GetUserTasksByStageBaseHandler<GetUserTasksExecutionCheckQuery>,
          IRequestHandler<GetUserTasksExecutionCheckQuery, BaseResponseDto<List<GetUserTasksResponse>>>
    {
        public GetUserTasksExecutionCheckQueryHandler(IMapper mapper, IUnitOfWork unitOfWork, IMemoryCache cache)
            : base(mapper, unitOfWork, cache, "ExecutionCheck") { }
    }
}
// ===========================================
// Примеры контроллер-эндпоинтов (по желанию)
// ===========================================
using BpmBaseApi.Shared.Queries.Process;
using BpmBaseApi.Shared.Responses.Process;
using BpmBaseApi.Shared.Dtos;
using MediatR;
using Microsoft.AspNetCore.Mvc;
using Swashbuckle.AspNetCore.Annotations;

[ApiController]
[Route("api/process")]
public class ProcessTasksByStageController : ControllerBase
{
    private readonly IMediator _mediator;
    public ProcessTasksByStageController(IMediator mediator) => _mediator = mediator;

    [HttpPost("tasks/approval")]
    [SwaggerResponse(StatusCodes.Status200OK, "OK", typeof(BaseResponseDto<List<GetUserTasksResponse>>))]
    public async Task<IActionResult> GetApproval([FromBody] GetUserTasksApprovalQuery q, CancellationToken ct)
        => Ok(await _mediator.Send(q, ct));

    [HttpPost("tasks/signing")]
    [SwaggerResponse(StatusCodes.Status200OK, "OK", typeof(BaseResponseDto<List<GetUserTasksResponse>>))]
    public async Task<IActionResult> GetSigning([FromBody] GetUserTasksSigningQuery q, CancellationToken ct)
        => Ok(await _mediator.Send(q, ct));

    [HttpPost("tasks/execution")]
    [SwaggerResponse(StatusCodes.Status200OK, "OK", typeof(BaseResponseDto<List<GetUserTasksResponse>>))]
    public async Task<IActionResult> GetExecution([FromBody] GetUserTasksExecutionQuery q, CancellationToken ct)
        => Ok(await _mediator.Send(q, ct));

    [HttpPost("tasks/execution-check")]
    [SwaggerResponse(StatusCodes.Status200OK, "OK", typeof(BaseResponseDto<List<GetUserTasksResponse>>))]
    public async Task<IActionResult> GetExecutionCheck([FromBody] GetUserTasksExecutionCheckQuery q, CancellationToken ct)
        => Ok(await _mediator.Send(q, ct));
}
