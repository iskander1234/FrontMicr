System.ArgumentException: GenericArguments[0], 'BpmBaseApi.Domain.Entities.Reference.DepartmentEntity', on 'BpmBaseApi.Persistence.Repositories.GenericRepositoryInt`1[TEntity]' violates the constraint of type 'TEntity'.
 ---> System.TypeLoadException: GenericArguments[0], 'BpmBaseApi.Domain.Entities.Reference.DepartmentEntity', on 'BpmBaseApi.Persistence.Repositories.GenericRepositoryInt`1[TEntity]' violates the constraint of type parameter 'TEntity'.
   at System.RuntimeTypeHandle.Instantiate(QCallTypeHandle handle, IntPtr* pInst, Int32 numGenericArgs, ObjectHandleOnStack type)
   at System.RuntimeType.MakeGenericType(Type[] instantiation)
   --- End of inner exception stack trace ---
   at System.RuntimeType.ValidateGenericArguments(MemberInfo definition, RuntimeType[] genericArguments, Exception e)
   at System.RuntimeType.MakeGenericType(Type[] instantiation)
   at Microsoft.Extensions.DependencyInjection.ServiceLookup.CallSiteFactory.TryCreateOpenGeneric(ServiceDescriptor descriptor, ServiceIdentifier serviceIdentifier, CallSiteChain callSiteChain, Int32 slot, Boolean throwOnConstraintViolation)
   at Microsoft.Extensions.DependencyInjection.ServiceLookup.CallSiteFactory.TryCreateOpenGeneric(ServiceIdentifier serviceIdentifier, CallSiteChain callSiteChain)
   at Microsoft.Extensions.DependencyInjection.ServiceLookup.CallSiteFactory.CreateCallSite(ServiceIdentifier serviceIdentifier, CallSiteChain callSiteChain)
   at Microsoft.Extensions.DependencyInjection.ServiceProvider.CreateServiceAccessor(ServiceIdentifier serviceIdentifier)
   at Microsoft.Extensions.DependencyInjection.ServiceProviderServiceExtensions.GetRequiredService(IServiceProvider provider, Type serviceType)
   at Microsoft.Extensions.DependencyInjection.ServiceProviderServiceExtensions.GetRequiredService[T](IServiceProvider provider)
   at BpmBaseApi.Persistence.UnitOfWork.get_DepartmentIntRepository() in C:\BPM\bpm\bpmbaseapi\BpmBaseApi.Persistence\UnitOfWork.cs:line 93
   at BpmBaseApi.Application.QueryHandlers.Common.GetEmployeeByLoginQueryHandler.Handle(GetEmployeeByLoginQuery query, CancellationToken cancellationToken) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi.Application\QueryHandlers\Common\GetEmployeeByLoginQueryHandler.cs:line 43
   at BpmBaseApi.Controllers.V1.EmployeeController.GetEmployee(String login) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi\Controllers\V1\EmployeeController.cs:line 43
   at Microsoft.AspNetCore.Mvc.Infrastructure.ActionMethodExecutor.TaskOfIActionResultExecutor.Execute(ActionContext actionContext, IActionResultTypeMapper mapper, ObjectMethodExecutor executor, Object controller, Object[] arguments)
   at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.<InvokeActionMethodAsync>g__Logged|12_1(ControllerActionInvoker invoker)
   at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.<InvokeNextActionFilterAsync>g__Awaited|10_0(ControllerActionInvoker invoker, Task lastTask, State next, Scope scope, Object state, Boolean isCompleted)
   at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.Rethrow(ActionExecutedContextSealed context)
   at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.Next(State& next, Scope& scope, Object& state, Boolean& isCompleted)
   at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.<InvokeInnerFilterAsync>g__Awaited|13_0(ControllerActionInvoker invoker, Task lastTask, State next, Scope scope, Object state, Boolean isCompleted)
   at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeFilterPipelineAsync>g__Awaited|20_0(ResourceInvoker invoker, Task lastTask, State next, Scope scope, Object state, Boolean isCompleted)
   at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeAsync>g__Logged|17_1(ResourceInvoker invoker)
   at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeAsync>g__Logged|17_1(ResourceInvoker invoker)
   at BpmBaseApi.Middlewares.ErrorHandlerMiddleware.Invoke(HttpContext context) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi\Middlewares\ErrorHandlerMiddleware.cs:line 24
   at BpmBaseApi.Middlewares.DefineUserHandler.Invoke(HttpContext context) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi\Middlewares\DefineUserHandler.cs:line 30
   at BpmBaseApi.Middlewares.CorrelationMiddleware.Invoke(HttpContext context) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi\Middlewares\CorrelationMiddleware.cs:line 16
   at Microsoft.AspNetCore.Authorization.AuthorizationMiddleware.Invoke(HttpContext context)
   at Microsoft.AspNetCore.Authentication.AuthenticationMiddleware.Invoke(HttpContext context)
   at Swashbuckle.AspNetCore.ReDoc.ReDocMiddleware.Invoke(HttpContext httpContext)
   at Swashbuckle.AspNetCore.SwaggerUI.SwaggerUIMiddleware.Invoke(HttpContext httpContext)
   at Swashbuckle.AspNetCore.Swagger.SwaggerMiddleware.Invoke(HttpContext httpContext, ISwaggerProvider swaggerProvider)
   at Microsoft.AspNetCore.Diagnostics.DeveloperExceptionPageMiddlewareImpl.Invoke(HttpContext context)

HEADERS
=======
Accept: text/plain; x-api-version=1.0
Connection: keep-alive
Host: localhost:5143
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/137.0.0.0 Safari/537.36
Accept-Encoding: gzip, deflate, br, zstd
Accept-Language: ru-RU,ru;q=0.9,en-US;q=0.8,en;q=0.7
Referer: http://localhost:5143/swagger/index.html
sec-ch-ua-platform: "Windows"
sec-ch-ua: "Google Chrome";v="137", "Chromium";v="137", "Not/A)Brand";v="24"
sec-ch-ua-mobile: ?0
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Correlation-Id: 59138a3e-29dc-4fed-9a16-9ac233210672


using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Dtos.Common;
using BpmBaseApi.Shared.Queries.Common;
using MediatR;
using Microsoft.AspNetCore.Mvc;

namespace BpmBaseApi.Controllers.V1;

[ApiController]
[Route("api/[controller]")]
public class EmployeeController : ControllerBase
{
    private readonly IMediator _mediator;

    public EmployeeController(IMediator mediator)
    {
        _mediator = mediator;
    }

    /// <summary>
    /// Получение информации о сотруднике по логину (login или loginAd)
    /// </summary>
    /// <param name="login">Логин или LoginAd сотрудника</param>
    /// <returns>Информация о сотруднике</returns>
    /// <response code="200">Успешно найден</response>
    /// <response code="400">Неверный запрос</response>
    /// <response code="404">Сотрудник не найден</response>
    [HttpGet]
    [ProducesResponseType(typeof(BaseResponseDto<EmployeeFullInfoDto>), 200)]
    [ProducesResponseType(typeof(BaseResponseDto<EmployeeFullInfoDto>), 400)]
    [ProducesResponseType(typeof(BaseResponseDto<EmployeeFullInfoDto>), 404)]
    public async Task<IActionResult> GetEmployee([FromQuery] string login)
    {
        if (string.IsNullOrWhiteSpace(login))
        {
            return BadRequest(new BaseResponseDto<EmployeeFullInfoDto>
            {
                Message = "Поле login обязательно",
                ErrorCode = 400
            });
        }

        var result = await _mediator.Send(new GetEmployeeByLoginQuery { Login = login });

        if (result.ErrorCode == 404)
            return NotFound(result);

        if (result.ErrorCode == 400)
            return BadRequest(result);

        return Ok(result);
    }  

}


using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Dtos.Common;
using BpmBaseApi.Shared.Queries.Common;
using MediatR;

namespace BpmBaseApi.Application.QueryHandlers.Common;

public class
    GetEmployeeByLoginQueryHandler : IRequestHandler<GetEmployeeByLoginQuery, BaseResponseDto<EmployeeFullInfoDto>>
{
    private readonly IUnitOfWork _unitOfWork;

    public GetEmployeeByLoginQueryHandler(IUnitOfWork unitOfWork)
    {
        _unitOfWork = unitOfWork;
    }

    public async Task<BaseResponseDto<EmployeeFullInfoDto>> Handle(GetEmployeeByLoginQuery query,
        CancellationToken cancellationToken)
    {
        if (string.IsNullOrWhiteSpace(query.Login))
        {
            return new BaseResponseDto<EmployeeFullInfoDto>
            {
                Message = "Login не должен быть пустым",
                ErrorCode = 400
            };
        }

        var employee = await _unitOfWork.EmployeeIntRepository
            .GetByFilterAsync(cancellationToken, e => e.Login == query.Login || e.LoginAd == query.Login);

        if (employee == null)
        {
            return new BaseResponseDto<EmployeeFullInfoDto>
            {
                Message = "Сотрудник не найден",
                ErrorCode = 404
            };
        }

        var department = await _unitOfWork.DepartmentIntRepository
            .GetByFilterAsync(cancellationToken, d => d.Id == employee.DepId);

        var parentDepartment = department?.ParentId != null
            ? await _unitOfWork.DepartmentIntRepository
                .GetByFilterAsync(cancellationToken, d => d.Id == department.ParentId)
            : null;

        return new BaseResponseDto<EmployeeFullInfoDto>
        {
            Data = new EmployeeFullInfoDto
            {
                Id = employee.Id,
                Name = employee.Name ?? "",
                Position = employee.Position ?? "",
                Login = employee.LoginAd ?? employee.Login ?? "",
                StatusCode = employee.StatusCode,
                StatusDescription = employee.StatusDescription ?? "",
                DepId = employee.DepId ?? "",
                DepName = employee.DepName ?? "",
                ParentDepId = department?.ParentId ?? "",
                ParentDepName = parentDepartment?.Name ?? "",
                IsFilial = employee.IsFilial,
                Mail = employee.Mail ?? "",
                LocalPhone = employee.LocalPhone ?? "",
                MobilePhone = employee.MobilePhone ?? "",
                IsManager = employee.IsManager,
                ManagerTabNumber = employee.ManagerTabNumber ?? "",
                Disabled = employee.Disabled,
                TabNumber = employee.TabNumber ?? ""
            }
        };
    }
}

using BpmBaseApi.Domain.Entities.Event.Employees;
using BpmBaseApi.Domain.SeedWork;

namespace BpmBaseApi.Domain.Entities.Employees;

/// <summary>
/// Сотрудник
/// </summary>
public class EmployeeEntity : BaseIntEntity
{
    public string Name { get; set; }
    public string? Position { get; set; }
    public string? Login { get; set; }
    public string? Mail { get; set; }
    public string? TabNumber { get; set; }
    public string? LocalPhone { get; set; }
    public string? MobilePhone { get; set; }
    public string? DepId { get; set; }
    public string? DepName { get; set; }
    public string? ParentDepId { get; set; }
    public string? ParentDepName { get; set; }
    public string? LoginAd { get; set; }
    public string? ManagerTabNumber { get; set; }
    public int? StatusCode { get; set; }
    public string? StatusDescription { get; set; }

    public bool Disabled { get; set; }
    public bool IsManager { get; set; }
    public bool IsFilial { get; set; }

    public virtual EmployeeEntity? ManagerTabNumberNavigation { get; set; }
    public virtual ICollection<EmployeeEntity> InverseManagerTabNumberNavigation { get; set; } = new List<EmployeeEntity>();

    public void Apply(EmployeeCreatedEvent @event)
    {
        Name = @event.Name;
        Position = @event.Position;
        Login = @event.Login;
        LoginAd = @event.LoginAd;
        StatusCode = @event.StatusCode;
        StatusDescription = @event.StatusDescription;
        DepId = @event.DepId;
        DepName = @event.DepName;
        IsFilial = @event.IsFilial;
        Mail = @event.Mail;
        LocalPhone = @event.LocalPhone;
        MobilePhone = @event.MobilePhone;
        IsManager = @event.IsManager;
        ManagerTabNumber = @event.ManagerTabNumber;
        Disabled = @event.Disabled;
        TabNumber = @event.TabNumber;
        ParentDepId = @event.ParentDepId;
        ParentDepName = @event.ParentDepName;
    }

}

using BpmBaseApi.Domain.Entities.AccessControl;
using BpmBaseApi.Domain.Entities.Common;
using BpmBaseApi.Domain.Entities.Employees;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Domain.Entities.Reference;

namespace BpmBaseApi.Persistence.Interfaces
{
    public interface IUnitOfWork
    {
        void Commit();
        // public ApplicationDbContext DbContext { get; }
        Task CommitAsync(CancellationToken cancellationToken);
        IJournaledGenericRepository<UserEntity> UserRepository { get; }
        IJournaledGenericRepository<RoleEntity> RoleRepository { get; }
        IJournaledGenericRepository<GroupEntity> GroupRepository { get; }
        IJournaledGenericRepository<UserGroupEntity> UserGroupRepository { get; }
        IJournaledGenericRepository<UserRoleEntity> UserRoleRepository { get; }
        IJournaledGenericRepository<GroupRoleEntity> GroupRoleRepository { get; }
        IJournaledGenericRepository<BlockEntity> BlockRepository { get; }
        IJournaledGenericRepository<ProcessDataEntity> ProcessDataRepository { get; }
        IJournaledGenericRepository<ProcessEntity> ProcessRepository { get; }
        IJournaledGenericRepository<ProcessTaskEntity> ProcessTaskRepository { get; }
        IJournaledGenericRepository<ProcessTaskHistoryEntity> ProcessTaskHistoryRepository { get; }
        //IJournaledGenericRepository<RequestSequenceEntity> RequestSequenceRepository { get; }
        IJournaledGenericRepository<RefProcessCategoryEntity> RefProcessCategoryRepository { get; }
        IJournaledGenericRepository<RefProcessEntity> RefProcessRepository { get; }
        IJournaledGenericRepository<RefInformationSystemEntity> RefInformationSystemRepository { get; }
        IJournaledGenericRepository<WorkSessionEntity> WorkSessionRepository { get; }
        IJournaledGenericRepository<WorkSessionLogEntity> WorkSessionLogRepository { get; }
        IJournaledGenericRepository<RequestNumberCounterEntity> RequestNumberCounterRepository { get; }
        IJournaledGenericRepository<MenuItemEntity> MenuItemRepository { get; }
        IJournaledGenericRepository<RoleMenuEntity> RoleMenuRepository { get; }
        IJournaledGenericRepository<RefBusinessObjectEntity> RefBusinessObjectRepository { get; }
        IJournaledGenericRepository<RefBusinessObjectAttributeEntity> RefBusinessObjectAttributeRepository { get; }
        IGenericRepositoryInt<EmployeeEntity> EmployeeIntRepository { get; }
        IGenericRepositoryInt<DepartmentEntity> DepartmentIntRepository { get; }

    }
}

