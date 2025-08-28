Надо написать 4 класса также из ProcessTaskEntity сравниваем  public string ProcessCode { get; set; } с      public enum ProcessStage
    {
        /// <summary>
        /// Согласование
        /// </summary>
        Approval,

        /// <summary>
        /// Подписание
        /// </summary>
        Signing,

        /// <summary>
        /// Исполнение
        /// </summary>
        Execution,

        /// <summary>
        /// Проверка исполнения
        /// </summary>
        ExecutionCheck,
 этими 4 нужно создать 4 класса как тот GetUserTasksQueryHandler но они должны называться GetUserTasksApprovalQueryHandler GetUserTasksSigningQueryHandler GetUserTasksExecutionQueryHandler GetUserTasksExecutionCheckQueryHandler using AutoMapper;
using BpmBaseApi.Domain.Entities.Process;
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
        var userCodeLower = query.UserCode.ToLower();

        // 1) Попытка взять из кэша
        if (cache.TryGetValue(cacheKey, out List<GetUserTasksResponse> tasksCache))
        {
            return new BaseResponseDto<List<GetUserTasksResponse>> { Data = tasksCache };
        }

        // 2) Всегда забираем свои (прямые) задачи Pending
        var ownTasks = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
            cancellationToken,
            p => p.AssigneeCode.ToLower() == userCodeLower && p.Status == "Pending");

        // 3) Проверяем, есть ли я в роли заместителя
        var asDeputy = await unitOfWork.DelegationRepository.GetByFilterListAsync(
            cancellationToken,
            d => d.DeputyUserCode.ToLower() == userCodeLower);

        List<ProcessTaskEntity> allTasksEntities = new List<ProcessTaskEntity>();
        allTasksEntities.AddRange(ownTasks);

        if (asDeputy.Any())
        {
            // 4) Собираем коды принципалов, кого я замещаю
            var principalCodes = asDeputy
                .Select(d => d.PrincipalUserCode.ToLower())
                .Distinct()
                .ToList();

            // 5) Забираем их Pending-задачи
            var principalTasks = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
                cancellationToken,
                p => principalCodes.Contains(p.AssigneeCode.ToLower()) && p.Status == "Pending");

            allTasksEntities.AddRange(principalTasks);
        }

        // 6) Мапим и кладём в кэш
        var responseList = mapper.Map<List<GetUserTasksResponse>>(allTasksEntities);
        cache.Set(cacheKey, responseList, TimeSpan.FromHours(1));

        return new BaseResponseDto<List<GetUserTasksResponse>> { Data = responseList };
    }
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
