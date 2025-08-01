"shortName": "Досгали И.Д.",

Мне надо сделать просто новое поле и shortName но в бд не надо просто выводит так сюда данный момент она такая 
{
  "data": {
    "id": 414,
    "name": "Досгали Искандер Досгалиұлы",
    "position": "Главный специалист",
    "login": "i.dosgali",
    "statusCode": 6,
    "statusDescription": "Работа",
    "depId": "19.100509",
    "depName": "Управление разработки Web приложений и сервисов",
    "parentDepId": "19.100500",
    "parentDepName": "Департамент цифровизации",
    "isFilial": false,
    "mail": "i.dosgali@enpf.kz",
    "localPhone": "0",
    "mobilePhone": "+7(747) 790-29-49",
    "isManager": false,
    "managerTabNumber": "4340",
    "disabled": false,
    "tabNumber": "00ЗП-00275"
  },
  "message": null,
  "errorCode": null
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
                ParentDepId = employee.ParentDepId ?? "",
                ParentDepName = employee.ParentDepName ?? "",
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

А нужно такой ответ и также у некоторых возможно нет очество просто тогда будет Досгали И.

{
  "data": {
    "id": 414,
    "name": "Досгали Искандер Досгалиұлы",
    "shortName": "Досгали И.Д.",
    "position": "Главный специалист",
    "login": "i.dosgali",
    "statusCode": 6,
    "statusDescription": "Работа",
    "depId": "19.100509",
    "depName": "Управление разработки Web приложений и сервисов",
    "parentDepId": "19.100500",
    "parentDepName": "Департамент цифровизации",
    "isFilial": false,
    "mail": "i.dosgali@enpf.kz",
    "localPhone": "0",
    "mobilePhone": "+7(747) 790-29-49",
    "isManager": false,
    "managerTabNumber": "4340",
    "disabled": false,
    "tabNumber": "00ЗП-00275"
  },
  "message": null,
  "errorCode": null
}
