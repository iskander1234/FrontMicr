Надо сделать так если ProcessTaskRepository AssigneeCode == system именно system не надо показывать  using AutoMapper;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Queries.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Application.QueryHandlers.Process
{
    public class GetProcessHistoryQueryHandler(
    IUnitOfWork unitOfWork,
    IMapper mapper,
    IProcessService processService
) : IRequestHandler<GetProcessHistoryQuery, BaseResponseDto<List<GetProcessHistoryResponse>>>
    {
        public async Task<BaseResponseDto<List<GetProcessHistoryResponse>>> Handle(GetProcessHistoryQuery query, CancellationToken cancellationToken)
        {
            var processTask = await unitOfWork.ProcessTaskRepository
                .GetByFilterAsync(cancellationToken, h => h.Id == query.TaskId) 
                              ?? throw new HandlerException("Задача не найдена", ErrorCodesEnum.Business);;
            
            var pdId = processTask.ProcessDataId;

            var history = await unitOfWork.ProcessTaskHistoryRepository
                .GetByFilterListAsync(cancellationToken, h => h.ProcessDataId == processTask.ProcessDataId);

            var historyItems = mapper.Map<List<GetProcessHistoryResponse>>(history.OrderBy(h => h.Timestamp));
            
            var tasks = await unitOfWork.ProcessTaskRepository
                .GetByFilterListAsync(cancellationToken, t => t.ProcessDataId == pdId);

            // Ручной маппинг активных задач в историю
            var taskItems = tasks.Select(t => new GetProcessHistoryResponse
            {
                
                TaskId       = t.Id,
                ParentTaskId = t.ParentTaskId,
                BlockCode    = t.BlockCode,
                BlockName    = t.BlockName,
                AssigneeCode = t.AssigneeCode,
                AssigneeName = t.AssigneeName,
                Comment      = t.Comment,
                Timestamp    = t.Created   
            }).ToList();

            var flat = historyItems
                .Concat(taskItems)
                .OrderBy(x => x.Timestamp)
                .ToList();

            var tree = processService.BuildHistoryTree(flat);

            return new BaseResponseDto<List<GetProcessHistoryResponse>> { Data = tree };
        }
        
     

    }
}
