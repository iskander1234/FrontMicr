 public async Task<Employee> GetEmployee(string login)
        {
            var ldapEmployee = await _bpmCoreContext
                .LdapEmployees
                .FirstOrDefaultAsync(x => x.Login.ToLower() == login.ToLower() || x.Email.ToLower() == (login + "@enpf.kz").ToLower());
            var splitedEmail = ldapEmployee?.Email?.Split("@")[0];
            var employee = await _bpmCoreContext.Employees.FirstOrDefaultAsync(x => x.Login == splitedEmail);
            if (employee == null)
                throw new Exception("Пользователя нет с таким логином");

            return employee;
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
