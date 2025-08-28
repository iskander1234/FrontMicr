using BpmBaseApi.Shared.Commands;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Queries.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Swashbuckle.AspNetCore.Annotations;
using static Microsoft.EntityFrameworkCore.DbLoggerCategory.Database;

namespace BpmBaseApi.Controllers.V1
{
    [Route("api/v{version:apiVersion}/[controller]")]
    [ApiController]
    [SwaggerTag("Сервис управления процессами")]
    public class ProcessController(IMediator mediator) : ControllerBase
    {
        /// <summary>
        /// Получение списка процессов
        /// </summary>
        /// <param name="query">Запрос на получение списка процессов</param>
        /// <param name="cancellationToken">Маркер отмены, используемый для отмены HTTP-запроса.</param>
        /// <returns></returns>
        [HttpGet("getprocesses")]
        [SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<List<GetProcessesResponse>>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        [SwaggerResponse(StatusCodes.Status503ServiceUnavailable)]
        public async Task<IActionResult> GetProcessesAsync(CancellationToken cancellationToken)
        {
            var query = new GetProcessesQuery();
            var result = await mediator.Send(query, cancellationToken);

            return Ok(result);
        }

        /// <summary>
        /// Запуск заявки
        /// </summary>
        /// <param name="command">Запрос для запуска заявки</param>
        /// <param name="cancellationToken">Маркер отмены, используемый для отмены HTTP-запроса.</param>
        /// <returns></returns>
        [HttpPost("start")]
        [SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<StartProcessResponse>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        [SwaggerResponse(StatusCodes.Status503ServiceUnavailable)]
        public async Task<IActionResult> StartProcessAsync([FromBody] StartProcessCommand command, CancellationToken cancellationToken)
        {
            var result = await mediator.Send(command, cancellationToken);

            return Ok(result);
        }

        /// <summary>
        /// Метод отправки заявки
        /// </summary>
        /// <param name="command">Запрос для отправки заявки</param>
        /// <param name="cancellationToken">Маркер отмены, используемый для отмены HTTP-запроса.</param>
        /// <returns></returns>
        [HttpPost("send")]
        [SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<SendProcessResponse>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        [SwaggerResponse(StatusCodes.Status503ServiceUnavailable)]
        public async Task<IActionResult> SendProcessAsync([FromBody] SendProcessCommand command, CancellationToken cancellationToken)
        {
            var result = await mediator.Send(command, cancellationToken);

            return Ok(result);
        }

        /// <summary>
        /// Метод получения заявок по пользователю
        /// </summary>
        /// <param name="query">Запрос получения заявок по пользователю</param>
        /// <param name="cancellationToken">Маркер отмены, используемый для отмены HTTP-запроса.</param>
        /// <returns></returns>
        [HttpPost("getusertasks")]
        [SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<List<GetUserTasksResponse>>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        [SwaggerResponse(StatusCodes.Status503ServiceUnavailable)]
        public async Task<IActionResult> GetUserTasksAsync([FromBody] GetUserTasksQuery query, CancellationToken cancellationToken)
        {
            var result = await mediator.Send(query, cancellationToken);

            return Ok(result);
        }

        /// <summary>
        /// Метод получения заявки по Id
        /// </summary>
        /// <param name="query">Запрос получения заявок по Id</param>
        /// <param name="cancellationToken">Маркер отмены, используемый для отмены HTTP-запроса.</param>
        /// <returns></returns>
        [HttpPost("getusertaskbyid")]
        [SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<GetUserTasksResponse>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        [SwaggerResponse(StatusCodes.Status503ServiceUnavailable)]
        public async Task<IActionResult> GetUserTaskByIdAsync([FromBody] GetUserTaskByIdQuery query, CancellationToken cancellationToken)
        {
            var result = await mediator.Send(query, cancellationToken);

            return Ok(result);
        }

        /// <summary>
        /// Метод получения истории заявки
        /// </summary>
        /// <param name="query">Запрос получения истории заявок</param>
        /// <param name="cancellationToken">Маркер отмены, используемый для отмены HTTP-запроса.</param>
        /// <returns></returns>
        [HttpPost("getprocesshistory")]
        [SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<List<GetProcessHistoryResponse>>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        [SwaggerResponse(StatusCodes.Status503ServiceUnavailable)]
        public async Task<IActionResult> GetProcessHistoryAsync([FromBody] GetProcessHistoryQuery query, CancellationToken cancellationToken)
        {
            var result = await mediator.Send(query, cancellationToken);

            return Ok(result);
        }

        /// <summary>
        /// Метод получения результаты согласования
        /// </summary>
        /// <param name="query">Запрос получения результаты согласования</param>
        /// <param name="cancellationToken">Маркер отмены, используемый для отмены HTTP-запроса.</param>
        /// <returns></returns>
        [HttpPost("getapprovalprocesshistory")]
        [SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<List<GetApprovalProcessHistoryResponse>>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        [SwaggerResponse(StatusCodes.Status503ServiceUnavailable)]
        public async Task<IActionResult> GetApprovalProcessHistoryAsync([FromBody] GetApprovalProcessHistoryQuery query, CancellationToken cancellationToken)
        {
            var result = await mediator.Send(query, cancellationToken);

            return Ok(result);
        }

        /// <summary>
        /// Метод получения заявок в которых участвовал пользователь
        /// </summary>
        /// <param name="query">Запрос получения </param>
        /// <param name="cancellationToken">Маркер отмены, используемый для отмены HTTP-запроса.</param>
        /// <returns></returns>
        [HttpPost("getuserrelatedprocesses")]
        [SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<List<GetUserRelatedProcessesResponse>>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        [SwaggerResponse(StatusCodes.Status503ServiceUnavailable)]
        public async Task<IActionResult> GetUserRelatedProcessesAsync([FromBody] GetUserRelatedProcessesQuery query, CancellationToken cancellationToken)
        {
            var result = await mediator.Send(query, cancellationToken);

            return Ok(result);
        }

        /// <summary>
        /// Метод завершает процесс и удаляет все связанные задачи
        /// </summary>
        /// <param name="command">Команда для завершения процесса и удаления всех связанных задач</param>
        /// <param name="cancellationToken">Маркер отмены, используемый для отмены HTTP-запроса.</param>
        /// <returns></returns>
        [HttpPost("completeprocess")]
        [SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<CompleteProcessResponse>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        [SwaggerResponse(StatusCodes.Status503ServiceUnavailable)]
        public async Task<IActionResult> CompleteProcessAsync([FromBody] CompleteProcessCommand command, CancellationToken cancellationToken)
        {
            var result = await mediator.Send(command, cancellationToken);

            return Ok(result);
        }

        /// <summary>
        /// Метод смены исполнителя
        /// </summary>
        /// <param name="command">Команда для смены исполнителя</param>
        /// <param name="cancellationToken">Маркер отмены, используемый для отмены HTTP-запроса.</param>
        /// <returns></returns>
        [HttpPost("reassignprocesstask")]
        [SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<ReassignProcessTaskResponse>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        [SwaggerResponse(StatusCodes.Status503ServiceUnavailable)]
        public async Task<IActionResult> ReassignProcessTaskAsync([FromBody] ReassignProcessTaskCommand command, CancellationToken cancellationToken)
        {
            var result = await mediator.Send(command, cancellationToken);

            return Ok(result);
        }

        /// <summary>
        ///     Проверяет есть ли согласующие
        /// </summary>
        /// <param name="command">Индивидуальный идентификатор процесса</param>
        /// <param name="cancellationToken">Маркер отмены, используемый для отмены HTTP-запроса</param>
        /// <returns></returns>
        [HttpPost("hasApprovers")]
        [SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<CheckHasApproversResponse>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        [SwaggerResponse(StatusCodes.Status503ServiceUnavailable)]
        public async Task<IActionResult> CheckHasApprovers([FromBody] CheckHasApproversCommand command, CancellationToken cancellationToken)
        {
            //var result = await mediator.Send(command, cancellationToken);

            //return Ok(result);
            return Ok(new BaseResponseDto<CheckHasApproversResponse>() { Data = new CheckHasApproversResponse() { Result = true } });
        }


        /// <summary>
        /// Установить этап процесса
        /// </summary>
        /// <param name="command">Индивидуальный идентификатор процесса</param>
        /// <param name="cancellationToken">Маркер отмены, используемый для отмены HTTP-запроса</param>
        /// <returns></returns>
        [HttpPost("setProcessStage")]
        [SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<SetProcessStageResponse>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        [SwaggerResponse(StatusCodes.Status503ServiceUnavailable)]
        public async Task<IActionResult> SetProcessStage([FromBody] SetProcessStageCommand command, CancellationToken cancellationToken)
        {
            var result = await mediator.Send(command, cancellationToken);

            return Ok(result);
        }

        /// <summary>
        /// Получить данные процесса через json
        /// </summary>
        /// <param name="query">Индивидуальный идентификатор процесса</param>
        /// <param name="cancellationToken">Маркер отмены, используемый для отмены HTTP-запроса</param>
        /// <returns></returns>
        [HttpPost("getProcessDataJson")]
        [SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<GetProcessDataJsonResponse>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        [SwaggerResponse(StatusCodes.Status503ServiceUnavailable)]
        public async Task<IActionResult> GetProcessDataJson([FromBody] GetProcessDataJsonQuery query, CancellationToken cancellationToken)
        {
            var result = await mediator.Send(query, cancellationToken);

            return Ok(result);
        }

        /// <summary>
        /// Получить канал отправки 
        /// </summary>
        /// <param name="command">Индивидуальный идентификатор процесса</param>
        /// <param name="cancellationToken">Маркер отмены, используемый для отмены HTTP-запроса</param>
        /// <returns></returns>
        [HttpPost("getSendingChannel")]
        [SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<GetSendingChannelResponse>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        [SwaggerResponse(StatusCodes.Status503ServiceUnavailable)]
        public async Task<IActionResult> GetSendingChannel([FromBody] GetSendingChannelCommand command, CancellationToken cancellationToken)
        {
            //Не до конца реализован
            //Надо создать обработчик и результат (класс)
            return Ok(new BaseResponseDto<GetSendingChannelResponse> { Data = new GetSendingChannelResponse() { Result = ResultChannel.Email } });
        }

        /// <summary>
        ///     Проверяет нужна ли перевод
        /// </summary>
        /// <param name="command">Индивидуальный идентификатор процесса</param>
        /// <param name="cancellationToken">Маркер отмены, используемый для отмены HTTP-запроса</param>
        /// <returns></returns>
        [HttpPost("needsTranslations")]
        [SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<NeedsTranslationResponse>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        [SwaggerResponse(StatusCodes.Status503ServiceUnavailable)]
        public async Task<IActionResult> CheckNeedTranslations([FromBody] NeedsTranslationCommand command, CancellationToken cancellationToken)
        {
            //Не до конца реализован
            //Надо создать обработчик и результат (класс)
            return Ok(new BaseResponseDto<NeedsTranslationResponse> { Data = new NeedsTranslationResponse() { Result = true } });
        }


        /// <summary>
        ///     
        /// </summary>
        /// <param name="command">Индивидуальный идентификатор процесса</param>
        /// <param name="cancellationToken">Маркер отмены, используемый для отмены HTTP-запроса</param>
        /// <returns></returns>
        [HttpPost("sendOutDocByChannel")]
        [SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<SendOutDocByChannelResponse>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        [SwaggerResponse(StatusCodes.Status503ServiceUnavailable)]
        public async Task<IActionResult> SendOutDocByChannel([FromBody] SendOutDocByChannelCommand command, CancellationToken cancellationToken)
        {
            //Не до конца реализован
            //Надо создать обработчик и результат (класс)
            //SendOutDocByChannelCommandHandler
            return Ok(new BaseResponseDto<SendOutDocByChannelResponse> { Data = new() { Status = "Ok" } });
        }


        /// <summary>
        ///     
        /// </summary>
        /// <param name="command">Индивидуальный идентификатор процесса</param>
        /// <param name="cancellationToken">Маркер отмены, используемый для отмены HTTP-запроса</param>
        /// <returns></returns>
        [HttpPost("getChangeClassification")]
        [SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<GetChangeClassificationResponse>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        [SwaggerResponse(StatusCodes.Status503ServiceUnavailable)]
        public async Task<IActionResult> GetChangeClassification([FromBody] GetChangeClassificationQuery command, CancellationToken cancellationToken)
        {
            //Не до конца реализован
            //Надо создать обработчик и результат (класс)
            //GetChangeClassificationQueryHandler
            return Ok(new BaseResponseDto<GetChangeClassificationResponse> { Data = new() { Result = ChangeClassification.Standard } });
        }
        
        
        
        [HttpPost("tasks/approval")]
        [SwaggerResponse(StatusCodes.Status200OK, "OK", typeof(BaseResponseDto<List<GetUserTasksResponse>>))]
        public async Task<IActionResult> GetApproval([FromBody] GetUserTasksQuery q, CancellationToken ct)
            => Ok(await mediator.Send(q, ct));

        [HttpPost("tasks/signing")]
        [SwaggerResponse(StatusCodes.Status200OK, "OK", typeof(BaseResponseDto<List<GetUserTasksResponse>>))]
        public async Task<IActionResult> GetSigning([FromBody] GetUserTasksQuery q, CancellationToken ct)
            => Ok(await mediator.Send(q, ct));

        [HttpPost("tasks/execution")]
        [SwaggerResponse(StatusCodes.Status200OK, "OK", typeof(BaseResponseDto<List<GetUserTasksResponse>>))]
        public async Task<IActionResult> GetExecution([FromBody] GetUserTasksQuery q, CancellationToken ct)
            => Ok(await mediator.Send(q, ct));

        [HttpPost("tasks/execution-check")]
        [SwaggerResponse(StatusCodes.Status200OK, "OK", typeof(BaseResponseDto<List<GetUserTasksResponse>>))]
        public async Task<IActionResult> GetExecutionCheck([FromBody] GetUserTasksQuery q, CancellationToken ct)
            => Ok(await mediator.Send(q, ct));
    }
}
using AutoMapper;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Queries.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using Microsoft.Extensions.Caching.Memory;

namespace BpmBaseApi.Application.QueryHandlers.Process
{
    public class GetUserTasksApprovalQueryHandler
        : GetUserTasksByStageBaseHandler<GetUserTasksQuery>,
            IRequestHandler<GetUserTasksQuery, BaseResponseDto<List<GetUserTasksResponse>>>
    {
        public GetUserTasksApprovalQueryHandler(IMapper mapper, IUnitOfWork unitOfWork, IMemoryCache cache)
            : base(mapper, unitOfWork, cache, "Approval") { }
    }

    public class GetUserTasksSigningQueryHandler
        : GetUserTasksByStageBaseHandler<GetUserTasksQuery>,
            IRequestHandler<GetUserTasksQuery, BaseResponseDto<List<GetUserTasksResponse>>>
    {
        public GetUserTasksSigningQueryHandler(IMapper mapper, IUnitOfWork unitOfWork, IMemoryCache cache)
            : base(mapper, unitOfWork, cache, "Signing") { }
    }

    public class GetUserTasksExecutionQueryHandler
        : GetUserTasksByStageBaseHandler<GetUserTasksQuery>,
            IRequestHandler<GetUserTasksQuery, BaseResponseDto<List<GetUserTasksResponse>>>
    {
        public GetUserTasksExecutionQueryHandler(IMapper mapper, IUnitOfWork unitOfWork, IMemoryCache cache)
            : base(mapper, unitOfWork, cache, "Execution") { }
    }

    public class GetUserTasksExecutionCheckQueryHandler
        : GetUserTasksByStageBaseHandler<GetUserTasksQuery>,
            IRequestHandler<GetUserTasksQuery, BaseResponseDto<List<GetUserTasksResponse>>>
    {
        public GetUserTasksExecutionCheckQueryHandler(IMapper mapper, IUnitOfWork unitOfWork, IMemoryCache cache)
            : base(mapper, unitOfWork, cache, "ExecutionCheck") { }
    }
}using AutoMapper;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using Microsoft.Extensions.Caching.Memory;

namespace BpmBaseApi.Application.QueryHandlers.Process;

public abstract class
    GetUserTasksByStageBaseHandler<TRequest> : IRequestHandler<TRequest, BaseResponseDto<List<GetUserTasksResponse>>>
    where TRequest : IRequest<BaseResponseDto<List<GetUserTasksResponse>>>
{
    private readonly IMapper _mapper;
    private readonly IUnitOfWork _unitOfWork;
    private readonly IMemoryCache _cache;
    private readonly string _stageCode; // "Approval" | "Signing" | "Execution" | "ExecutionCheck"

    protected GetUserTasksByStageBaseHandler(
        IMapper mapper,
        IUnitOfWork unitOfWork,
        IMemoryCache cache,
        string stageCode)
    {
        _mapper = mapper;
        _unitOfWork = unitOfWork;
        _cache = cache;
        _stageCode = stageCode;
    }

    // Достаём UserCode через рефлексию, чтобы не дублировать код
    private static string GetUserCodeFromRequest(TRequest request)
    {
        var prop = typeof(TRequest).GetProperty("UserCode");
        if (prop == null)
            throw new InvalidOperationException($"{typeof(TRequest).Name} должен иметь свойство UserCode");
        return (string)(prop.GetValue(request) ?? throw new InvalidOperationException("UserCode is null"));
    }

    public async Task<BaseResponseDto<List<GetUserTasksResponse>>> Handle(
        TRequest request,
        CancellationToken cancellationToken)
    {
        var userCode = GetUserCodeFromRequest(request);
        var userCodeLower = userCode.ToLowerInvariant();
        var cacheKey = $"tasks:{_stageCode}:{userCodeLower}";

        if (_cache.TryGetValue(cacheKey, out List<GetUserTasksResponse> cached))
            return new BaseResponseDto<List<GetUserTasksResponse>> { Data = cached };

        // 1) Свои Pending задачи по конкретному этапу
        var ownTasks = await _unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
            cancellationToken,
            p => p.AssigneeCode.ToLower() == userCodeLower
                 && p.Status == "Pending"
                 && p.BlockCode == _stageCode);

        var all = new List<ProcessTaskEntity>(ownTasks);

        // 2) Я — заместитель?
        var asDeputy = await _unitOfWork.DelegationRepository.GetByFilterListAsync(
            cancellationToken,
            d => d.DeputyUserCode.ToLower() == userCodeLower);

        if (asDeputy.Any())
        {
            var principalCodes = asDeputy
                .Select(d => d.PrincipalUserCode.ToLowerInvariant())
                .Distinct()
                .ToList();

            var principalTasks = await _unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
                cancellationToken,
                p => principalCodes.Contains(p.AssigneeCode.ToLower())
                     && p.Status == "Pending"
                     && p.BlockCode == _stageCode);

            all.AddRange(principalTasks);
        }

        var response = _mapper.Map<List<GetUserTasksResponse>>(all);
        _cache.Set(cacheKey, response, TimeSpan.FromHours(1));

        return new BaseResponseDto<List<GetUserTasksResponse>> { Data = response };
    }
}
