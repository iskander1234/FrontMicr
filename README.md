using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Persistence.Interfaces;
using Microsoft.EntityFrameworkCore;

namespace BpmBaseApi.Persistence.Repositories;

public class DelegationRepository
    : JournaledGenericRepository<DelegationEntity>, IDelegationRepository
{
    public DelegationRepository(ApplicationDbContext ctx) : base(ctx) { }

    public Task<List<DelegationEntity>> GetByDeputyAsync(
        string deputyCode,
        CancellationToken ct) =>
        dbSet
            .Where(d => d.DeputyUserCode.ToLower() == deputyCode.ToLower())
            .ToListAsync(ct);
}

using BpmBaseApi.Domain.Entities.Process;

namespace BpmBaseApi.Persistence.Interfaces;

/// <summary>
/// Репозиторий для сущности делегаций, расширяющий общий CRUD из IJournaledGenericRepository
/// </summary>
public interface IDelegationRepository : IJournaledGenericRepository<DelegationEntity>
{
    /// <summary>
    /// Возвращает все делегации, где заданный код — заместитель
    /// </summary>
    Task<List<DelegationEntity>> GetByDeputyAsync(string deputyCode, CancellationToken ct);
}


using AutoMapper;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Queries.Process;
using BpmBaseApi.Shared.Responses.Process;
using BpmBaseApi.Shared.Responses.Reference;
using MediatR;
using Microsoft.Extensions.Caching.Memory;
using static Microsoft.EntityFrameworkCore.DbLoggerCategory.Database;

namespace BpmBaseApi.Application.QueryHandlers.Process
{
    
    public class GetUserTasksQueryHandler(
        IMapper mapper,
        IUnitOfWork unitOfWork,
        IMemoryCache cache
    ) : IRequestHandler<GetUserTasksQuery, BaseResponseDto<List<GetUserTasksResponse>>>
    {
        public async Task<BaseResponseDto<List<GetUserTasksResponse>>> Handle(
            GetUserTasksQuery query,
            CancellationToken cancellationToken)
        {
            string cacheKey = $"tasks:{query.UserCode}";

            // 1) Сначала пробуем взять из кэша
            if (cache.TryGetValue(cacheKey, out List<GetUserTasksResponse> tasksCache))
            {
                return new BaseResponseDto<List<GetUserTasksResponse>> { Data = tasksCache };
            }

            // 2) Получаем список кодов «принципалов», которых я замещаю
            var delegations = await unitOfWork.DelegationRepository
                .GetByDeputyAsync(query.UserCode, cancellationToken);

            var principalCodes = delegations
                .Select(d => d.PrincipalUserCode)
                .Distinct()
                .ToList();

            // 3) Выбираем задачи:
            //    – либо где assignee == я (query.UserCode)
            //    – либо где assignee == любой из principalCodes
            //    и при этом статус == "Pending"
            var tasks = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
                cancellationToken,
                p =>
                    (p.AssigneeCode == query.UserCode ||
                     principalCodes.Contains(p.AssigneeCode))
                    && p.Status == "Pending"
            );

            // 4) Мапим в DTO и сохраняем в кэш на час
            var responseList = mapper.Map<List<GetUserTasksResponse>>(tasks);
            cache.Set(cacheKey, responseList, TimeSpan.FromHours(1));

            // 5) Возвращаем результат
            return new BaseResponseDto<List<GetUserTasksResponse>> { Data = responseList };
        }
    }
   
}
