namespace BpmBaseApi.Shared.Dtos.Pagination;

public class PaginationMeta
{
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalItems { get; set; }
    public int TotalPages { get; set; }
    public bool HasPrevious => Page > 1;
    public bool HasNext => Page < TotalPages;

    public PaginationMeta() { }
    public PaginationMeta(int totalItems, int page, int pageSize)
    {
        TotalItems = totalItems;
        Page = page < 1 ? 1 : page;
        PageSize = pageSize < 1 ? 1 : pageSize;
        TotalPages = (int)Math.Ceiling(totalItems / (double)PageSize);
        if (TotalPages == 0) TotalPages = 1;
        if (Page > TotalPages) Page = TotalPages;
    }
}


using BpmBaseApi.Shared.Dtos;

namespace BpmBaseApi.Shared.Dtos.Pagination;

public class PagedResponseDto<TItem> : BaseResponseDto<List<TItem>>
{
    public PaginationMeta Pagination { get; set; } = new();
}


namespace BpmBaseApi.Shared.Queries.Common;

public abstract class PagedQuery
{
    public int Page { get; set; } = 1;
    public int PageSize { get; set; } = 50;
}


namespace BpmBaseApi.Shared.Queries.Common;

public static class Paging
{
    public const int DefaultPage = 1;
    public const int DefaultPageSize = 50;
    public const int MaxPageSize = 200;

    public static (int page, int size) Normalize(int page, int pageSize)
    {
        var p = page <= 0 ? DefaultPage : page;
        var s = pageSize <= 0 ? DefaultPageSize : pageSize;
        if (s > MaxPageSize) s = MaxPageSize;
        return (p, s);
    }
}


namespace BpmBaseApi.Application.Common.Extensions;

public static class PaginationExtensions
{
    public static IEnumerable<T> ToPage<T>(this IEnumerable<T> src, int page, int pageSize)
        => src.Skip((page - 1) * pageSize).Take(pageSize);
}



БЫЛО

public class GetDepartmentWorkTimeQuery : IRequest<BaseResponseDto<List<DepartmentWorkTimeResponse>>>
{
    public string DepartmentId { get; set; } = "";
    public PeriodType PeriodType { get; set; }
    public DateOnly? Date { get; set; }
    public int Year { get; set; }
    public int Month { get; set; }
}

СТАЛО

using BpmBaseApi.Shared.Queries.Common;

public class GetDepartmentWorkTimeQuery 
    : PagedQuery, // ← добавили базовый класс (НЕ интерфейс)
      IRequest<BaseResponseDto<List<DepartmentWorkTimeResponse>>>
{
    public string DepartmentId { get; set; } = "";
    public PeriodType PeriodType { get; set; }
    public DateOnly? Date { get; set; }
    public int Year { get; set; }
    public int Month { get; set; }

    // поля Page и PageSize уже есть в PagedQuery
}


using System.Globalization;
using BpmBaseApi.Application.Common.Extensions;     // + пагинация
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Dtos.Pagination;          // + PagedResponseDto, PaginationMeta
using BpmBaseApi.Shared.Queries.Common;
using BpmBaseApi.Shared.Responses.Common;
using MediatR;

namespace BpmBaseApi.Application.QueryHandlers.Common;

public class GetDepartmentWorkTimeQueryHandler 
    : IRequestHandler<GetDepartmentWorkTimeQuery, BaseResponseDto<List<DepartmentWorkTimeResponse>>> // ← тип не меняем
{
    private readonly IUnitOfWork _unitOfWork;

    public GetDepartmentWorkTimeQueryHandler(IUnitOfWork unitOfWork)
    {
        _unitOfWork = unitOfWork;
    }

    public async Task<BaseResponseDto<List<DepartmentWorkTimeResponse>>> Handle(
        GetDepartmentWorkTimeQuery query, CancellationToken cancellationToken)
    {
        // ★ нормализуем локально (не пишем обратно в query)
        var (page, pageSize) = Paging.Normalize(query.Page, query.PageSize); // ★

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

        // ——— СОРТИРОВКУ/ПРОЕКЦИЮ НЕ МЕНЯЕМ ———
        var culture = CultureInfo.GetCultureInfo("ru-RU");

        var full = sessions
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
                    WeekDay = culture.DateTimeFormat.GetDayName(s.StartTime.DayOfWeek),
                    Status = s.TotalTime.HasValue && s.TotalTime.Value >= TimeSpan.FromHours(9) ? "OK" : ""
                };
            })
            .ToList();

        var total = full.Count;                         // ★ всего записей
        var pageItems = full.ToPage(page, pageSize)     // ★ режем страницу (in-memory)
                          .ToList();

        // ВОЗВРАЩАЕМ тип как раньше (BaseResponseDto<List<...>>),
        // но фактически объект — PagedResponseDto (наследник) с полем pagination.
        return new PagedResponseDto<DepartmentWorkTimeResponse> // ★
        {
            Data = pageItems,
            Pagination = new PaginationMeta(total, page, pageSize)
        };
    }

    private string FormatTimeSpan(TimeSpan? time)
    {
        if (!time.HasValue) return "";
        return $"{(int)time.Value.TotalHours:D2}:{time.Value.Minutes:D2}";
    }
}

