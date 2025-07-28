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
        public async Task<BaseResponseDto<List<GetUserTasksResponse>>> Handle(GetUserTasksQuery query, CancellationToken cancellationToken)
        {
            string cacheKey = $"tasks:{query.UserCode}";

            var tasksCache = cache.Get<List<GetUserTasksResponse>>(cacheKey);

            if (tasksCache == null)
            {
                var tasks = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
                    cancellationToken,
                    p => p.AssigneeCode == query.UserCode && p.Status == "Pending");

                var responseList = mapper.Map<List<GetUserTasksResponse>>(tasks);

                cache.Set(cacheKey, responseList, TimeSpan.FromHours(1));

                return new BaseResponseDto<List<GetUserTasksResponse>> { Data = responseList };
            }

            return new BaseResponseDto<List<GetUserTasksResponse>> { Data = tasksCache };
        }
    }
}
