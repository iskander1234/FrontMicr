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
