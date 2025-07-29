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

    var loginLower = query.Login.Trim().ToLower();

    // 1) Сначала ищем запись в LDAP-таблице:
    //    Login.ToLower() == loginLower
    //    ИЛИ Email.ToLower() == $"{loginLower}@enpf.kz"
    var ldapEmployee = await _unitOfWork.LdapEmployeeRepository
        .GetByFilterAsync(ct, x =>
            x.Login.ToLower() == loginLower
         || x.Email.ToLower() == $"{loginLower}@enpf.kz");

    // 2) Если нашли — берём часть до '@', иначе оставляем исходный login
    var actualLogin = ldapEmployee is not null && 
                      !string.IsNullOrWhiteSpace(ldapEmployee.Email)
        ? ldapEmployee.Email.Split('@')[0]
        : loginLower;

    // 3) Ищем сотрудника по actualLogin в основной таблице:
    //    e.Login == actualLogin  ИЛИ e.LoginAd == actualLogin
    var employee = await _unitOfWork.EmployeeIntRepository
        .GetByFilterAsync(ct, e =>
            e.Login   == actualLogin
         || e.LoginAd == actualLogin);

    if (employee == null)
        return new BaseResponseDto<EmployeeFullInfoDto>
        {
            Message   = "Сотрудник не найден",
            ErrorCode = 404
        };

    // 4) Получаем департамент и родительский департамент
    var department = await _unitOfWork.DepartmentIntRepository
        .GetByFilterAsync(ct, d => d.Id == employee.DepId);

    var parentDepartment = department?.ParentId != null
        ? await _unitOfWork.DepartmentIntRepository
            .GetByFilterAsync(ct, d => d.Id == department.ParentId)
        : null;

    // 5) Мапим в DTO и возвращаем
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
