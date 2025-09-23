using AutoMapper;
using BpmBaseApi.Application.QueryHandlers.Process;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Queries.ITSM;
using BpmBaseApi.Shared.Queries.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using Microsoft.Extensions.Caching.Memory;

namespace BpmBaseApi.Application.QueryHandlers.ITSM
{
    public class GetITSMUserTasksQueryHandler
        (
            IUnitOfWork unitOfWork,
            IMapper mapper,
            IMemoryCache cache
        )
        : IRequestHandler<GetITSMUserTasksQuery, BaseResponseDto<List<GetUserTasksResponse>>>
    {
        public async Task<BaseResponseDto<List<GetUserTasksResponse>>> Handle(GetITSMUserTasksQuery request, CancellationToken cancellationToken)
        {
            var userLower = request.UserCode.ToLowerInvariant();
            var cacheKey = $"itsm_tasks:{request.BlockCode}:{request.UserCode}";

            if (cache.TryGetValue(cacheKey, out List<GetUserTasksResponse> data))
                return new() { Data = data };

            var own = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
                cancellationToken,
                p => p.AssigneeCode != null
                     && p.AssigneeCode.ToLower() == userLower
                     && p.Status == "Pending"
                     && p.BlockCode == request.BlockCode.ToString()
            );

            var all = new List<ProcessTaskEntity>(own);

            var asDeputy = await unitOfWork.DelegationRepository.GetByFilterListAsync(
                cancellationToken,
                d => d.DeputyUserCode != null && d.DeputyUserCode.ToLower() == userLower
            );

            if (asDeputy.Any())
            {
                var principals = asDeputy
                    .Where(d => d.PrincipalUserCode != null)
                    .Select(d => d.PrincipalUserCode!.ToLower())
                    .Distinct()
                    .ToList();

                var principalTasks = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
                    cancellationToken,
                    p => p.AssigneeCode != null
                         && principals.Contains(p.AssigneeCode.ToLower())
                         && p.Status == "Pending"
                         && p.BlockCode == request.BlockCode.ToString()
                );

                all.AddRange(principalTasks);
            }

            var ordered = all
                .GroupBy(t => t.Id)
                .Select(g => g.First())
                .OrderByDescending(t => t.Created)
                .ToList();

            var response = mapper.Map<List<GetUserTasksResponse>>(ordered);
            cache.Set(cacheKey, response, TimeSpan.FromHours(1));
            return new() { Data = response };
        }
    }
}
