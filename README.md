запросы в бд так но у меня не правильно SELECT * FROM public."ProcessTask"
WHERE "Id" = 'd40056fc-f9e9-47d5-9095-e434ee77553b'


select * from public."ProcessTaskHistory"
WHERE "ProcessDataId" = '79af5959-bc77-4dc1-8571-9350c23c390b' and "BlockCode" = 'Execution' 



using AutoMapper;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Enum;
using BpmBaseApi.Shared.Queries.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Application.QueryHandlers.Process;


public class GetExecutionProcessHistoryQueryHandler(
    IUnitOfWork unitOfWork,
    IMapper mapper,
    IProcessService processService) :
    IRequestHandler<GetExecutionProcessHistoryQuery, BaseResponseDto<List<GetExecutionProcessHistoryResponse>>>
{
    public async Task<BaseResponseDto<List<GetExecutionProcessHistoryResponse>>> Handle(GetExecutionProcessHistoryQuery query, CancellationToken cancellationToken)
    {
        var processTask = await unitOfWork.ProcessTaskRepository
            .GetByFilterAsync(cancellationToken, h => h.Id == query.TaskId);

        var history = await unitOfWork.ProcessTaskHistoryRepository
            .GetByFilterListAsync
            (
                cancellationToken,
                h => h.ProcessDataId == processTask.ProcessDataId && h.BlockCode == ProcessStage.Execution.ToString()
            );

        var historyItems = mapper.Map<List<GetProcessHistoryResponse>>(history.OrderBy(h => h.Timestamp));

        var tree = processService.BuildHistoryTree(historyItems);

        return new BaseResponseDto<List<GetExecutionProcessHistoryResponse>>
        {
            Data = mapper.Map<List<GetExecutionProcessHistoryResponse>>(tree)
        };
    }
}


using AutoMapper;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Enum;
using BpmBaseApi.Shared.Queries.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Application.QueryHandlers.Process;

public class GetExecutionProcessDataHistoryQueryHandler(
    IUnitOfWork unitOfWork,
    IMapper mapper,
    IProcessService processService) :
    IRequestHandler<GetExecutionProcessDataHistoryQuery, BaseResponseDto<List<GetExecutionProcessHistoryResponse>>>
{
    public async Task<BaseResponseDto<List<GetExecutionProcessHistoryResponse>>> Handle(GetExecutionProcessDataHistoryQuery query, CancellationToken cancellationToken)
    {
        
        var history = await unitOfWork.ProcessTaskHistoryRepository
            .GetByFilterListAsync
            (
                cancellationToken,
                h => h.ProcessDataId == query.ProcessDataId && h.BlockCode == ProcessStage.Execution.ToString()
            );

        var historyItems = mapper.Map<List<GetProcessHistoryResponse>>(history.OrderBy(h => h.Timestamp));

        var tree = processService.BuildHistoryTree(historyItems);

        return new BaseResponseDto<List<GetExecutionProcessHistoryResponse>>
        {
            Data = mapper.Map<List<GetExecutionProcessHistoryResponse>>(tree)
        };
    }
}


using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Shared.Queries.Process
{
    public class GetExecutionProcessHistoryQuery : IRequest<BaseResponseDto<List<GetExecutionProcessHistoryResponse>>>
    {
        /// <summary>
        /// Уникальный идентификатор 
        /// </summary>
        public Guid TaskId { get; set; }
    }
}


using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Shared.Queries.Process
{
    public class GetExecutionProcessDataHistoryQuery : IRequest<BaseResponseDto<List<GetExecutionProcessHistoryResponse>>>
    {
        /// <summary>
        /// Уникальный идентификатор 
        /// </summary>
        public Guid ProcessDataId { get; set; }
    }
}

namespace BpmBaseApi.Shared.Responses.Process;

public abstract class GetExecutionProcessHistoryResponse : GetProcessHistoryResponse
{
    
}


namespace BpmBaseApi.Shared.Responses.Process
{
    public class GetProcessHistoryResponse
    {
        public Guid TaskId { get; set; }
        public Guid? ParentTaskId { get; set; }
        public string BlockCode { get; set; }
        public string BlockName { get; set; }
        public string AssigneeCode { get; set; }
        public string AssigneeName { get; set; }
        public string Action { get; set; }
        public string? Comment { get; set; }
        //public string? Condition { get; set; }
        public DateTime Timestamp { get; set; }
        public string? ActionName { get; set; }
        public List<GetProcessHistoryResponse> Children { get; set; } = new();
    }
}


