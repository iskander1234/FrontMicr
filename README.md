// ========================== GetEmployeeSubordinatesHandler ==========================
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
        .Select(emp =>
        {
            var shortName = GetShortName(emp.Name);

            return new GetEmployeeSubordinatesResponse
            {
                Id = emp.Id,
                Name = emp.Name,
                ShortName = shortName, // <---- добавили
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
            };
        })
        .ToList();

    return new BaseResponseDto<List<GetEmployeeSubordinatesResponse>> { Data = result };
}


// ========================== GetSimilarEmployeesHandler ==========================
public async Task<BaseResponseDto<List<GetSimilarEmployeesResponse>>> Handle(GetSimilarEmployeesQuery request, CancellationToken cancellationToken)
{
    var employees = await _unitOfWork.EmployeeIntRepository.GetByFilterListAsync(
        cancellationToken,
        a => a.Name.ToLower().Contains(request.FullName.ToLower())
    );

    var result = employees
        .Select(emp =>
        {
            var shortName = GetShortName(emp.Name);

            return new GetSimilarEmployeesResponse
            {
                Id = emp.Id,
                Name = emp.Name,
                ShortName = shortName, // <---- добавили
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
            };
        })
        .ToList();

    return new BaseResponseDto<List<GetSimilarEmployeesResponse>> { Data = result };
}

