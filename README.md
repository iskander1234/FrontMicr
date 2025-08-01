// Shared/Dtos/Common/EmployeeFullInfoDto.cs
namespace BpmBaseApi.Shared.Dtos.Common
{
    public class EmployeeFullInfoDto
    {
        public int    Id               { get; set; }
        public string Name             { get; set; }
        /// <summary>
        /// Фамилия и инициалы: «Фамилия И.О.» или «Фамилия И.»
        /// </summary>
        public string ShortName        { get; set; }
        public string Position         { get; set; }
        public string Login            { get; set; }
        public int    StatusCode       { get; set; }
        public string StatusDescription{ get; set; }
        public string DepId            { get; set; }
        public string DepName          { get; set; }
        public string ParentDepId      { get; set; }
        public string ParentDepName    { get; set; }
        public bool   IsFilial         { get; set; }
        public string Mail             { get; set; }
        public string LocalPhone       { get; set; }
        public string MobilePhone      { get; set; }
        public bool   IsManager        { get; set; }
        public string ManagerTabNumber { get; set; }
        public bool   Disabled         { get; set; }
        public string TabNumber        { get; set; }
    }
}

// Application/QueryHandlers/Common/GetEmployeeByLoginQueryHandler.cs
using BpmBaseApi.Shared.Dtos.Common;
// ...

public async Task<BaseResponseDto<EmployeeFullInfoDto>> Handle(
    GetEmployeeByLoginQuery query,
    CancellationToken cancellationToken)
{
    // ... ваша валидация и поиск employee ...

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
