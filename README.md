public class GetUserTasksApprovalQuery      : IRequest<BaseResponseDto<List<GetUserTasksResponse>>> { public string UserCode { get; set; } = default!; }
public class GetUserTasksSigningQuery       : IRequest<BaseResponseDto<List<GetUserTasksResponse>>> { public string UserCode { get; set; } = default!; }
public class GetUserTasksExecutionQuery     : IRequest<BaseResponseDto<List<GetUserTasksResponse>>> { public string UserCode { get; set; } = default!; }
public class GetUserTasksExecutionCheckQuery: IRequest<BaseResponseDto<List<GetUserTasksResponse>>> { public string UserCode { get; set; } = default!; }


public class GetUserTasksApprovalQueryHandler
  : GetUserTasksByStageBaseHandler<GetUserTasksApprovalQuery>,
    IRequestHandler<GetUserTasksApprovalQuery, BaseResponseDto<List<GetUserTasksResponse>>>
{
    public GetUserTasksApprovalQueryHandler(IMapper m, IUnitOfWork u, IMemoryCache c)
        : base(m, u, c, "Approval") {}
}

public class GetUserTasksSigningQueryHandler
  : GetUserTasksByStageBaseHandler<GetUserTasksSigningQuery>,
    IRequestHandler<GetUserTasksSigningQuery, BaseResponseDto<List<GetUserTasksResponse>>>
{
    public GetUserTasksSigningQueryHandler(IMapper m, IUnitOfWork u, IMemoryCache c)
        : base(m, u, c, "Signing") {}
}

public class GetUserTasksExecutionQueryHandler
  : GetUserTasksByStageBaseHandler<GetUserTasksExecutionQuery>,
    IRequestHandler<GetUserTasksExecutionQuery, BaseResponseDto<List<GetUserTasksResponse>>>
{
    public GetUserTasksExecutionQueryHandler(IMapper m, IUnitOfWork u, IMemoryCache c)
        : base(m, u, c, "Execution") {}
}

public class GetUserTasksExecutionCheckQueryHandler
  : GetUserTasksByStageBaseHandler<GetUserTasksExecutionCheckQuery>,
    IRequestHandler<GetUserTasksExecutionCheckQuery, BaseResponseDto<List<GetUserTasksResponse>>>
{
    public GetUserTasksExecutionCheckQueryHandler(IMapper m, IUnitOfWork u, IMemoryCache c)
        : base(m, u, c, "ExecutionCheck") {}
}
