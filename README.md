using AutoMapper;
using BpmBaseApi.Domain.Entities.Employees;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Queries.Common;
using MediatR;

namespace BpmBaseApi.Application.QueryHandlers.Common
{
    public class GetEmployeeByLoginQueryHandler
        : IRequestHandler<GetEmployeeByLoginQuery, BaseResponseDto<EmployeeFullInfoDto>>
    {
        private readonly IUnitOfWork _uow;
        private readonly ILdapEmployeeRepository _ldapRepo;

        public GetEmployeeByLoginQueryHandler(
            IUnitOfWork unitOfWork,
            ILdapEmployeeRepository ldapRepo)
        {
            _uow     = unitOfWork;
            _ldapRepo = ldapRepo;
        }

        public async Task<BaseResponseDto<EmployeeFullInfoDto>> Handle(
            GetEmployeeByLoginQuery query,
            CancellationToken ct)
        {
            if (string.IsNullOrWhiteSpace(query.Login))
                return new BaseResponseDto<EmployeeFullInfoDto>
                {
                    Message   = "Login не должен быть пустым",
                    ErrorCode = 400
                };

            // 1) Попытка найти в LDAP
            var loginLower = query.Login.ToLower();
            var ldap = await _ldapRepo.GetByFilterAsync(ct,
                x => x.Login.ToLower() == loginLower
                   || x.Email.ToLower() == $"{loginLower}@enpf.kz");

            string actualLoginForEmployee = query.Login;

            if (ldap != null && !string.IsNullOrWhiteSpace(ldap.Email))
            {
                // убираем доменную часть, чтобы получить login в таблице сотрудников
                actualLoginForEmployee = ldap.Email.Split('@')[0];
            }

            // 2) Ищем сотрудника по логину или по LoginAd
            var employee = await _uow.EmployeeIntRepository.GetByFilterAsync(ct,
                e => e.Login == actualLoginForEmployee
                  || e.LoginAd == actualLoginForEmployee);

            if (employee == null)
                return new BaseResponseDto<EmployeeFullInfoDto>
                {
                    Message   = "Сотрудник не найден",
                    ErrorCode = 404
                };

            // 3) Забираем подразделение и родительское подразделение
            var department = await _uow.DepartmentIntRepository.GetByFilterAsync(ct,
                d => d.Id == employee.DepId);

            var parentDepartment = department?.ParentId != null
                ? await _uow.DepartmentIntRepository.GetByFilterAsync(ct,
                    d => d.Id == department.ParentId)
                : null;

            // 4) Мапим в DTO и возвращаем
            var dto = new EmployeeFullInfoDto
            {
                Id               = employee.Id,
                Name             = employee.Name ?? "",
                Position         = employee.Position ?? "",
                Login            = employee.LoginAd ?? employee.Login ?? "",
                StatusCode       = employee.StatusCode,
                StatusDescription= employee.StatusDescription ?? "",
                DepId            = employee.DepId ?? "",
                DepName          = employee.DepName ?? "",
                ParentDepId      = department?.ParentId ?? "",
                ParentDepName    = parentDepartment?.Name ?? "",
                IsFilial         = employee.IsFilial,
                Mail             = employee.Mail ?? "",
                LocalPhone       = employee.LocalPhone ?? "",
                MobilePhone      = employee.MobilePhone ?? "",
                IsManager        = employee.IsManager,
                ManagerTabNumber = employee.ManagerTabNumber ?? "",
                Disabled         = employee.Disabled,
                TabNumber        = employee.TabNumber ?? ""
            };

            return new BaseResponseDto<EmployeeFullInfoDto> { Data = dto };
        }
    }
}
