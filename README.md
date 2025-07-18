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
   at Microsoft.Extensions.DependencyInjection.ServiceLookup.CallSiteFactory.GetCallSite(ServiceIdentifier serviceIdentifier, CallSiteChain callSiteChain)
   at Microsoft.Extensions.DependencyInjection.ServiceProvider.CreateServiceAccessor(ServiceIdentifier serviceIdentifier)
   at Microsoft.Extensions.DependencyInjection.ServiceProviderServiceExtensions.GetRequiredService(IServiceProvider provider, Type serviceType)
   at Microsoft.Extensions.DependencyInjection.ServiceProviderServiceExtensions.GetRequiredService[T](IServiceProvider provider)
   at BpmBaseApi.Persistence.UnitOfWork.get_DepartmentIntRepository() in C:\BPM\bpm\bpmbaseapi\BpmBaseApi.Persistence\UnitOfWork.cs:line 93
   at BpmBaseApi.Application.QueryHandlers.Common.GetDepartmentWorkTimeQueryHandler.Handle(GetDepartmentWorkTimeQuery query, CancellationToken cancellationToken) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi.Application\QueryHandlers\Common\GetDepartmentWorkTimeQueryHandler.cs:line 59
   at BpmBaseApi.Controllers.V1.WorkTimeController.GetByDepartment(GetDepartmentWorkTimeQuery query, CancellationToken cancellationToken) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi\Controllers\V1\WorkTimeController.cs:line 110
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
Content-Type: application/json; x-api-version=1.0
Origin: http://localhost:5143
Referer: http://localhost:5143/swagger/index.html
Content-Length: 112
sec-ch-ua-platform: "Windows"
sec-ch-ua: "Google Chrome";v="137", "Chromium";v="137", "Not/A)Brand";v="24"
sec-ch-ua-mobile: ?0
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Correlation-Id: 052c76f5-69d1-43ca-a72f-30be472d2cb8

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


using BpmBaseApi.Domain.Entities.Event.Reference;
using BpmBaseApi.Domain.SeedWork;

namespace BpmBaseApi.Domain.Entities.Reference;

/// <summary>
/// Департамент / отдел
/// </summary>
public class DepartmentEntity 
{
    public string Id { get; set; } = null!;
    public string Name { get; set; }
    public string? ManagerId { get; set; }
    public int Actual { get; set; }

    public string? ParentId { get; set; }

    public virtual DepartmentEntity? Parent { get; set; }
    public virtual ICollection<DepartmentEntity> InverseParent { get; set; } = new List<DepartmentEntity>();

    public virtual ICollection<DepartmentCreatedEvent> CreatedEvents { get; set; } =
        new List<DepartmentCreatedEvent>();

    public void Apply(DepartmentCreatedEvent @event)
    {
        Id = @event.Id;
        ParentId = @event.ParentId;
        Name = @event.Name;
        ManagerId = @event.ManagerId;
        Actual = @event.Actual;
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


using System.Globalization;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Queries.Common;
using BpmBaseApi.Shared.Responses.Common;
using MediatR;

namespace BpmBaseApi.Application.QueryHandlers.Common;

public class GetDepartmentWorkTimeQueryHandler : IRequestHandler<GetDepartmentWorkTimeQuery, BaseResponseDto<List<DepartmentWorkTimeResponse>>>
{
    private readonly IUnitOfWork _unitOfWork;

    public GetDepartmentWorkTimeQueryHandler(IUnitOfWork unitOfWork)
    {
        _unitOfWork = unitOfWork;
    }

    public async Task<BaseResponseDto<List<DepartmentWorkTimeResponse>>> Handle(GetDepartmentWorkTimeQuery query, CancellationToken cancellationToken)
    {
        Console.WriteLine($"[Debug] DepartmetId {query.DepartmentId}");
        Console.WriteLine($"[Debug] Periodtype {query.PeriodType}, Date {query.Date}, Year {query.Year}, Month {query.Month}");

        DateTime startDate;
        DateTime endDate;

        if (query.PeriodType == PeriodType.Day && query.Date.HasValue)
        {
            startDate = query.Date.Value.ToDateTime(TimeOnly.MinValue);
            endDate = query.Date.Value.ToDateTime(TimeOnly.MaxValue);
        }
        else
        {
            startDate = new DateTime(query.Year, query.Month, 1);
            endDate = startDate.AddMonths(1).AddDays(-1);
        }

        Console.WriteLine($"[Debug] StartDate {startDate}, EndDate {endDate}");

        var sessions = await _unitOfWork.WorkSessionRepository.GetByFilterListAsync(
            cancellationToken,
            s => s.ParentDepId == query.DepartmentId &&
                 s.StartTime >= startDate &&
                 s.StartTime <= endDate);

        Console.WriteLine($"[Debug] Сессий найденно {sessions.Count}");

        foreach (var s in sessions.Take(5))
        {
            Console.WriteLine($"[Debug] Session {s.UserCode}, {s.UserName}, {s.StartTime} - {s.EndTime}");
        }

        var userCodes = sessions.Select(s => s.UserCode).Distinct().ToList();

        var employees = await _unitOfWork.EmployeeIntRepository.GetByFilterListAsync(
            cancellationToken,
            e => userCodes.Contains(e.LoginAd));

        var departments = await _unitOfWork.DepartmentIntRepository.GetByFilterListAsync(cancellationToken);

        var employeeMap = employees.ToDictionary(
            e => e.LoginAd,
            e =>
            {
                var dep = departments.FirstOrDefault(d => d.Id.Trim() == (e.DepId?.Trim() ?? ""));
                var parentName = departments.FirstOrDefault(d => d.Id.Trim() == (dep?.ParentId?.Trim() ?? ""))?.Name ?? "";
                return (
                    e.Position ?? "",
                    e.DepName ?? "",
                    parentName
                );
            });

        Console.WriteLine($"[Debug] Employee map count {employeeMap.Count}");
        foreach (var emp in employeeMap.Take(5))
        {
            Console.WriteLine($"[Debug] {emp.Key} => {emp.Value.Item1}, {emp.Value.Item2}, {emp.Value.Item3}");
        }

        var result = sessions
            .OrderBy(s =>
            {
                if (employeeMap.TryGetValue(s.UserCode, out var info))
                    return info.Item2;
                return "";
            })
            .ThenBy(s => s.UserName)
            .ThenBy(s => s.StartTime)
            .Select(s =>
            {
                var info = employeeMap.TryGetValue(s.UserCode, out var data)
                    ? data
                    : ("", "", "");

                return new DepartmentWorkTimeResponse
                {
                    UserCode = s.UserCode,
                    FullName = s.UserName,
                    Position = info.Item1,
                    DepartmentName = info.Item2,
                    ParentDepartmentName = info.Item3,
                    Date = s.StartTime.ToString("dd.MM.yyyy"),
                    StartTime = s.StartTime.ToString("HH:mm"),
                    EndTime = s.EndTime?.ToString("HH:mm"),
                    TotalTime = FormatTimeSpan(s.TotalTime),
                    WeekDay = CultureInfo.GetCultureInfo("ru-RU").DateTimeFormat.GetDayName(s.StartTime.DayOfWeek),
                    Status = s.TotalTime.HasValue && s.TotalTime.Value >= TimeSpan.FromHours(9) ? "OK" : ""
                };
            })
            .ToList();

        return new BaseResponseDto<List<DepartmentWorkTimeResponse>> { Data = result };
    }

    private string FormatTimeSpan(TimeSpan? time)
    {
        if (!time.HasValue) return "";
        return $"{(int)time.Value.TotalHours:D2}:{time.Value.Minutes:D2}";
    }
}

