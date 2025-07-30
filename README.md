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

            if (cache.TryGetValue(cacheKey, out List<GetUserTasksResponse> tasksCache))
                return new BaseResponseDto<List<GetUserTasksResponse>> { Data = tasksCache };

            // 1) Получаем все делегации, где я — заместитель, через дженерик-метод
            var delegations = await unitOfWork.DelegationRepository
                .GetByFilterListAsync(cancellationToken,
                    d => d.DeputyUserCode.ToLower() == query.UserCode.ToLower());


            var principalCodes = delegations
                .Select(d => d.PrincipalUserCode)
                .Distinct()
                .ToList();

            // 2) Берём задачи и для меня, и для принципалов
            var tasks = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
                cancellationToken,
                p => (p.AssigneeCode == query.UserCode
                      || principalCodes.Contains(p.AssigneeCode))
                     && p.Status == "Pending");

            var responseList = mapper.Map<List<GetUserTasksResponse>>(tasks);
            cache.Set(cacheKey, responseList, TimeSpan.FromHours(1));

            return new BaseResponseDto<List<GetUserTasksResponse>> { Data = responseList };
        }
    }
