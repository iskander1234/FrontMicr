using AutoMapper;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Models.Process;
using BpmBaseApi.Shared.Queries.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using Microsoft.Extensions.Caching.Memory;

namespace BpmBaseApi.Application.QueryHandlers.Process
{
    public class GetUserTaskByIdQueryHandler(
        IMapper mapper,
        IUnitOfWork unitOfWork
        ) : IRequestHandler<GetUserTaskByIdQuery, BaseResponseDto<GetUserTasksResponse>>
    {
        public async Task<BaseResponseDto<GetUserTasksResponse>> Handle(GetUserTaskByIdQuery query, CancellationToken cancellationToken)
        {
            var task = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(cancellationToken, p => p.Id == query.TaskId)
                ?? throw new HandlerException("Запрос с таким идентификатором не найден", ErrorCodesEnum.Business);

            var taskFiles = await unitOfWork
                .ProcessFileRepository
                .GetByFilterListAsync
                (
                    cancellationToken,
                    p => p.ProcessDataId == task.ProcessDataId
                );

            var response = mapper.Map<GetUserTasksResponse>(task);
            response.Files = mapper.Map<List<FileDto>>(taskFiles);

            return new BaseResponseDto<GetUserTasksResponse> { Data = response };
        }
    }
}
