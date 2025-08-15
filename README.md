Это пример тут есть  shortName и мне нужно так же shortName добавить using BpmBaseApi.Persistence.Interfaces;
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

        var fullName = employee.Name ?? "";
        // Разбиваем по пробелу: [0]=Фамилия, [1]=Имя, [2]=Отчество (если есть)
        var parts = fullName
            .Split(' ', StringSplitOptions.RemoveEmptyEntries);

        string shortName;
        if (parts.Length == 0)
        {
            shortName = "";
        }
        else if (parts.Length == 1)
        {
            // Только фамилия
            shortName = parts[0];
        }
        else
        {
            // Фамилия + инициалы
            var sb = new System.Text.StringBuilder();
            sb.Append(parts[0]);                    // Фамилия
            sb.Append(' ');
            sb.Append(char.ToUpper(parts[1][0]));   // первая буква имени
            sb.Append('.');
            if (parts.Length >= 3 && !string.IsNullOrWhiteSpace(parts[2]))
            {
                sb.Append(char.ToUpper(parts[2][0])); // первая буква отчества
                sb.Append('.');
            }
            shortName = sb.ToString();
        }

        var dto = new EmployeeFullInfoDto
        {
            Id               = employee.Id,
            Name             = fullName,
            ShortName        = shortName,
            Position         = employee.Position ?? "",
            Login            = employee.LoginAd ?? employee.Login ?? "",
            StatusCode       = employee.StatusCode,
            StatusDescription= employee.StatusDescription ?? "",
            DepId            = employee.DepId ?? "",
            DepName          = employee.DepName ?? "",
            ParentDepId      = employee.ParentDepId ?? "",
            ParentDepName    = employee.ParentDepName ?? "",
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
}  добавить shortName нужно сюда этим 2 методам handle   GetSimilarEmployeesHandler и GetEmployeeSubordinatesHandler  using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Queries.Employee;
using BpmBaseApi.Shared.Responses.Employee;
using MediatR;

namespace BpmBaseApi.Application.QueryHandlers.Employees
{
    public class GetSimilarEmployeesHandler : IRequestHandler<GetSimilarEmployeesQuery, BaseResponseDto<List<GetSimilarEmployeesResponse>>>
    {
        private readonly IUnitOfWork _unitOfWork;

        public GetSimilarEmployeesHandler(IUnitOfWork unitOfWork)
        {
            _unitOfWork = unitOfWork;
        }

        public async Task<BaseResponseDto<List<GetSimilarEmployeesResponse>>> Handle(GetSimilarEmployeesQuery request, CancellationToken cancellationToken)
        {
            var employees = await _unitOfWork.EmployeeIntRepository.GetByFilterListAsync(
                cancellationToken,
                a => a.Name.ToLower().Contains(request.FullName.ToLower())
            );

            var result = employees
                .Select(emp => new GetSimilarEmployeesResponse
                {
                    Id = emp.Id,
                    Name = emp.Name,
                    Position = emp.Position,
                    Login = emp.Login,
                    LoginAD = emp.LoginAd,
                    StatusCode = emp.StatusCode,
                    StatusDescription = emp.StatusDescription,
                    DepId = emp.DepId,
                    DepName = emp.DepName,
                    ParentDepId = emp.ParentDepId,
                    ParentDepName = emp.ParentDepName,
                    IsFilial = emp.IsFilial,
                    Mail = emp.Mail,
                    LocalPhone = emp.LocalPhone,
                    MobilePhone = emp.MobilePhone,
                    IsManager = emp.IsManager,
                    ManagerTabNumber = emp.ManagerTabNumber,
                    Disabled = emp.Disabled,
                    TabNumber = emp.TabNumber
                })
                .ToList();

            return new BaseResponseDto<List<GetSimilarEmployeesResponse>> { Data = result };
        }
    }
}
using BpmBaseApi.Domain.Entities.Employees;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Queries.Employee;
using BpmBaseApi.Shared.Responses.Employee;
using MediatR;

namespace BpmBaseApi.Application.QueryHandlers.Employees
{
    public class GetEmployeeSubordinatesHandler : IRequestHandler<GetEmployeeSubordinatesQuery, BaseResponseDto<List<GetEmployeeSubordinatesResponse>>>
    {
        private readonly IUnitOfWork _unitOfWork;

        public GetEmployeeSubordinatesHandler(IUnitOfWork unitOfWork)
        {
            _unitOfWork = unitOfWork;
        }

        public async Task<BaseResponseDto<List<GetEmployeeSubordinatesResponse>>> Handle(GetEmployeeSubordinatesQuery request, CancellationToken cancellationToken)
        {
            var employees = new List<EmployeeEntity>();

            if (request.Employee.Login == "z.kurmanov")
                employees = await _unitOfWork.EmployeeIntRepository.GetByFilterListAsync(
                    cancellationToken,
                    a => a.Login != "z.kurmanov" && a.DepId == "01.001010"
                    );

            if (request.Employee.TabNumber == "3253")
                employees = await _unitOfWork.EmployeeIntRepository.GetByFilterListAsync(
                    cancellationToken,
                    a => a.ManagerTabNumber == "3253" && a.Position!.ToLower().Contains("директор")
                    );

            if (request.Employee.TabNumber == "3616")
                employees = [];

            if (request.Employee.Position.ToLower() == "управляющий директор" || request.Employee.Position.ToLower() == "заместитель председателя правления, член правления")
                employees = await _unitOfWork.EmployeeIntRepository.GetByFilterListAsync(
                    cancellationToken,
                    a => a.ManagerTabNumber == request.Employee.TabNumber
                    );

            if (request.Employee.Position.ToLower() == "директор департамента" || request.Employee.Position.ToLower() == "директор филиала")
                employees = await _unitOfWork.EmployeeIntRepository.GetByFilterListAsync(
                    cancellationToken,
                    a => a.ManagerTabNumber == request.Employee.TabNumber
                        && (a.Position!.ToLower().Contains("начальник") || a.Position!.ToLower().Contains("заместитель"))
                    );

            if (request.Employee.Position.ToLower().Contains("начальник"))
                employees = await _unitOfWork.EmployeeIntRepository.GetByFilterListAsync(
                    cancellationToken,
                    a => a.DepId == request.Employee.DepId && a.IsManager == false
                    );

            var result = employees
                .Select(
                    emp => new GetEmployeeSubordinatesResponse
                    {
                        Id = emp.Id,
                        Name = emp.Name,
                        Position = emp.Position,
                        Login = emp.Login,
                        LoginAD = emp.LoginAd,
                        StatusCode = emp.StatusCode,
                        StatusDescription = emp.StatusDescription,
                        DepId = emp.DepId,
                        DepName = emp.DepName,
                        ParentDepId = emp.ParentDepId,
                        ParentDepName = emp.ParentDepName,
                        IsFilial = emp.IsFilial,
                        Mail = emp.Mail,
                        LocalPhone = emp.LocalPhone,
                        MobilePhone = emp.MobilePhone,
                        IsManager = emp.IsManager,
                        ManagerTabNumber = emp.ManagerTabNumber,
                        Disabled = emp.Disabled,
                        TabNumber = emp.TabNumber
                    }
                )
                .ToList();

            return new BaseResponseDto<List<GetEmployeeSubordinatesResponse>> { Data = result };
        }
    }
}
