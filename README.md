System.InvalidOperationException: The LINQ expression 'DbSet<ProcessTaskEntity>()
    .Where(p => p.AssigneeCode != null && p.AssigneeCode.ToLower() == __userLower_0 && p.Status == "Pending" && p.BlockCode != null && string.Equals(
        a: p.BlockCode, 
        b: ___stageCode_1, 
        comparisonType: Ordinal))' could not be translated. Additional information: Translation of the 'string.Equals' overload with a 'StringComparison' parameter is not supported. See https://go.microsoft.com/fwlink/?linkid=2129535 for more information. Either rewrite the query in a form that can be translated, or switch to client evaluation explicitly by inserting a call to 'AsEnumerable', 'AsAsyncEnumerable', 'ToList', or 'ToListAsync'. See https://go.microsoft.com/fwlink/?linkid=2101038 for more information.
   at Microsoft.EntityFrameworkCore.Query.QueryableMethodTranslatingExpressionVisitor.Translate(Expression expression)
   at Microsoft.EntityFrameworkCore.Query.QueryCompilationContext.CreateQueryExecutorExpression[TResult](Expression query)
   at Microsoft.EntityFrameworkCore.Query.QueryCompilationContext.CreateQueryExecutor[TResult](Expression query)
   at Microsoft.EntityFrameworkCore.Storage.Database.CompileQuery[TResult](Expression query, Boolean async)
   at Microsoft.EntityFrameworkCore.Query.Internal.QueryCompiler.CompileQueryCore[TResult](IDatabase database, Expression query, IModel model, Boolean async)
   at Microsoft.EntityFrameworkCore.Query.Internal.QueryCompiler.<>c__DisplayClass11_0`1.<ExecuteCore>b__0()
   at Microsoft.EntityFrameworkCore.Query.Internal.CompiledQueryCache.GetOrAddQuery[TResult](Object cacheKey, Func`1 compiler)
   at Microsoft.EntityFrameworkCore.Query.Internal.QueryCompiler.ExecuteCore[TResult](Expression query, Boolean async, CancellationToken cancellationToken)
   at Microsoft.EntityFrameworkCore.Query.Internal.QueryCompiler.ExecuteAsync[TResult](Expression query, CancellationToken cancellationToken)
   at Microsoft.EntityFrameworkCore.Query.Internal.EntityQueryProvider.ExecuteAsync[TResult](Expression expression, CancellationToken cancellationToken)
   at Microsoft.EntityFrameworkCore.Query.Internal.EntityQueryable`1.GetAsyncEnumerator(CancellationToken cancellationToken)
   at System.Runtime.CompilerServices.ConfiguredCancelableAsyncEnumerable`1.GetAsyncEnumerator()
   at Microsoft.EntityFrameworkCore.EntityFrameworkQueryableExtensions.ToListAsync[TSource](IQueryable`1 source, CancellationToken cancellationToken)
   at BpmBaseApi.Persistence.Repositories.JournaledGenericRepository`1.GetByFilterListAsync(CancellationToken cancellationToken, Expression`1 filter, Func`2 include, Func`2 orderBy) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi.Persistence\Repositories\JournaledGenericRepository.cs:line 55
   at BpmBaseApi.Application.QueryHandlers.Process.GetUserTasksByStageBaseHandler`1.Handle(TRequest request, CancellationToken ct) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi.Application\QueryHandlers\Process\GetUserTasksByStageBaseHandler.cs:line 37
   at BpmBaseApi.Controllers.V1.ProcessController.GetSigning(GetUserTasksQuery q, CancellationToken ct) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi\Controllers\V1\ProcessController.cs:line 375
   at Microsoft.AspNetCore.Mvc.Infrastructure.ActionMethodExecutor.TaskOfIActionResultExecutor.Execute(ActionContext actionContext, IActionResultTypeMapper mapper, ObjectMethodExecutor executor, Object controller, Object[] arguments)
   at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.<InvokeActionMethodAsync>g__Logged|12_1(ControllerActionInvoker invoker)
   at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.<InvokeNextActionFilterAsync>g__Awaited|10_0(ControllerActionInvoker invoker, Task lastTask, State next, Scope scope, Object state, Boolean isCompleted)
   at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.Rethrow(ActionExecutedContextSealed context)
   at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.Next(State& next, Scope& scope, Object& state, Boolean& isCompleted)
   at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.InvokeInnerFilterAsync()
--- End of stack trace from previous location ---
   at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeFilterPipelineAsync>g__Awaited|20_0(ResourceInvoker invoker, Task lastTask, State next, Scope scope, Object state, Boolean isCompleted)
   at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeAsync>g__Logged|17_1(ResourceInvoker invoker)
   at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeAsync>g__Logged|17_1(ResourceInvoker invoker)
   at BpmBaseApi.Middlewares.ErrorHandlerMiddleware.Invoke(HttpContext context) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi\Middlewares\ErrorHandlerMiddleware.cs:line 24
   at BpmBaseApi.Middlewares.DefineUserHandler.Invoke(HttpContext context) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi\Middlewares\DefineUserHandler.cs:line 30
   at BpmBaseApi.Middlewares.CorrelationMiddleware.Invoke(HttpContext context) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi\Middlewares\CorrelationMiddleware.cs:line 16
   at Microsoft.AspNetCore.Authorization.AuthorizationMiddleware.Invoke(HttpContext context)
   at Microsoft.AspNetCore.Authentication.AuthenticationMiddleware.Invoke(HttpContext context)
   at Swashbuckle.AspNetCore.ReDoc.ReDocMiddleware.Invoke(HttpContext httpContext)
   at Swashbuckle.AspNetCore.SwaggerUI.SwaggerUIMiddleware.Invoke(HttpContext httpContext)
   at Swashbuckle.AspNetCore.Swagger.SwaggerMiddleware.Invoke(HttpContext httpContext, ISwaggerProvider swaggerProvider)
   at Microsoft.AspNetCore.Diagnostics.DeveloperExceptionPageMiddlewareImpl.Invoke(HttpContext context)

HEADERS
=======
Accept: */*
Connection: keep-alive
Host: localhost:5143
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/137.0.0.0 Safari/537.36
Accept-Encoding: gzip, deflate, br, zstd
Accept-Language: ru-RU,ru;q=0.9,en-US;q=0.8,en;q=0.7
Content-Type: application/json; x-api-version=1.0
Origin: http://localhost:5143
Referer: http://localhost:5143/swagger/index.html
Content-Length: 31
sec-ch-ua-platform: "Windows"
sec-ch-ua: "Google Chrome";v="137", "Chromium";v="137", "Not/A)Brand";v="24"
sec-ch-ua-mobile: ?0
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Correlation-Id: d3e684ca-404a-40a1-874c-b01158d4ae66 




using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Shared.Queries.Process
{
    public class GetUserTasksQuery : IRequest<BaseResponseDto<List<GetUserTasksResponse>>>
    {
        /// <summary>
        /// Код пользователя
        /// </summary>
        public string UserCode { get; set; } 
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

        using AutoMapper;
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

}
 
