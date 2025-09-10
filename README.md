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


/// <summary>
        /// Метод получения результаты исполнение по taskId
        /// </summary>
        /// <param name="query">Запрос получения результаты согласования</param>
        /// <param name="cancellationToken">Маркер отмены, используемый для отмены HTTP-запроса.</param>
        /// <returns></returns>
        [HttpPost("getexecutionprocesstaskidhistorytask")]
        [SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<List<GetExecutionProcessHistoryResponse>>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        [SwaggerResponse(StatusCodes.Status503ServiceUnavailable)]
        public async Task<IActionResult> GetExecutionProcessTaskIdHistoryAsync([FromBody] GetExecutionProcessHistoryQuery query, CancellationToken cancellationToken)
        {
            var result = await mediator.Send(query, cancellationToken);

            return Ok(result);
        }
        
        /// <summary>
        /// Метод получения результаты исполнение по dataId
        /// </summary>
        /// <param name="query">Запрос получения результаты согласования</param>
        /// <param name="cancellationToken">Маркер отмены, используемый для отмены HTTP-запроса.</param>
        /// <returns></returns>
        [HttpPost("getexecutionprocessdataidhistorytask")]
        [SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<List<GetExecutionProcessHistoryResponse>>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        [SwaggerResponse(StatusCodes.Status503ServiceUnavailable)]
        public async Task<IActionResult> GetExecutionProcessProcessDataIdHistoryAsync([FromBody] GetExecutionProcessDataHistoryQuery query, CancellationToken cancellationToken)
        {
            var result = await mediator.Send(query, cancellationToken);

            return Ok(result);
        }
