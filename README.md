using AutoMapper;
using BpmBaseApi.Persistence;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Queries.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Application.QueryHandlers.Process
{
    public class GetUserRelatedProcessesQueryHandler(
        IMapper mapper,
        IUnitOfWork unitOfWork
        ) : IRequestHandler<GetUserRelatedProcessesQuery, BaseResponseDto<List<GetUserRelatedProcessesResponse>>>
    {
        public async Task<BaseResponseDto<List<GetUserRelatedProcessesResponse>>> Handle(GetUserRelatedProcessesQuery query, CancellationToken cancellationToken)
        {
            var history = await unitOfWork.ProcessTaskHistoryRepository.GetByFilterListAsync(cancellationToken,
            h => h.AssigneeCode == query.UserCode && h.ProcessCode == query.ProcessCode);

            var processDataIds = history
            .Select(h => h.ProcessDataId)
            .Distinct()
            .ToList();

            var processes = await unitOfWork.ProcessDataRepository.GetByFilterListAsync(cancellationToken,
            p => processDataIds.Contains(p.Id));

            var mapped = mapper.Map<List<GetUserRelatedProcessesResponse>>(processes);

            return new BaseResponseDto<List<GetUserRelatedProcessesResponse>> { Data = mapped };
        }
    }
}
