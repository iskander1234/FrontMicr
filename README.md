Microsoft.EntityFrameworkCore.DbUpdateException: An error occurred while saving the entity changes. See the inner exception for details.
 ---> Npgsql.PostgresException (0x80004005): 23502: null value in column "AssigneeName" of relation "ProcessTask" violates not-null constraint

DETAIL: Detail redacted as it may contain sensitive data. Specify 'Include Error Detail' in the connection string to include this information.
   at Npgsql.Internal.NpgsqlConnector.ReadMessageLong(Boolean async, DataRowLoadingMode dataRowLoadingMode, Boolean readingNotifications, Boolean isReadingPrependedMessage)
   at System.Runtime.CompilerServices.PoolingAsyncValueTaskMethodBuilder`1.StateMachineBox`1.System.Threading.Tasks.Sources.IValueTaskSource<TResult>.GetResult(Int16 token)
   at Npgsql.NpgsqlDataReader.NextResult(Boolean async, Boolean isConsuming, CancellationToken cancellationToken)
   at Npgsql.NpgsqlDataReader.NextResult(Boolean async, Boolean isConsuming, CancellationToken cancellationToken)
   at Npgsql.EntityFrameworkCore.PostgreSQL.Update.Internal.NpgsqlModificationCommandBatch.Consume(RelationalDataReader reader, Boolean async, CancellationToken cancellationToken)
  Exception data:
    Severity: ERROR
    SqlState: 23502
    MessageText: null value in column "AssigneeName" of relation "ProcessTask" violates not-null constraint
    Detail: Detail redacted as it may contain sensitive data. Specify 'Include Error Detail' in the connection string to include this information.
    SchemaName: public
    TableName: ProcessTask
    ColumnName: AssigneeName
    File: execMain.c
    Line: 2009
    Routine: ExecConstraints
   --- End of inner exception stack trace ---
   at Npgsql.EntityFrameworkCore.PostgreSQL.Update.Internal.NpgsqlModificationCommandBatch.Consume(RelationalDataReader reader, Boolean async, CancellationToken cancellationToken)
   at Microsoft.EntityFrameworkCore.Update.ReaderModificationCommandBatch.ExecuteAsync(IRelationalConnection connection, CancellationToken cancellationToken)
   at Microsoft.EntityFrameworkCore.Update.ReaderModificationCommandBatch.ExecuteAsync(IRelationalConnection connection, CancellationToken cancellationToken)
   at Microsoft.EntityFrameworkCore.Update.Internal.BatchExecutor.ExecuteAsync(IEnumerable`1 commandBatches, IRelationalConnection connection, CancellationToken cancellationToken)
   at Microsoft.EntityFrameworkCore.Update.Internal.BatchExecutor.ExecuteAsync(IEnumerable`1 commandBatches, IRelationalConnection connection, CancellationToken cancellationToken)
   at Microsoft.EntityFrameworkCore.Update.Internal.BatchExecutor.ExecuteAsync(IEnumerable`1 commandBatches, IRelationalConnection connection, CancellationToken cancellationToken)
   at Microsoft.EntityFrameworkCore.Storage.RelationalDatabase.SaveChangesAsync(IList`1 entries, CancellationToken cancellationToken)
   at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.StateManager.SaveChangesAsync(IList`1 entriesToSave, CancellationToken cancellationToken)
   at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.StateManager.SaveChangesAsync(StateManager stateManager, Boolean acceptAllChangesOnSuccess, CancellationToken cancellationToken)
   at Npgsql.EntityFrameworkCore.PostgreSQL.Storage.Internal.NpgsqlExecutionStrategy.ExecuteAsync[TState,TResult](TState state, Func`4 operation, Func`4 verifySucceeded, CancellationToken cancellationToken)
   at Microsoft.EntityFrameworkCore.DbContext.SaveChangesAsync(Boolean acceptAllChangesOnSuccess, CancellationToken cancellationToken)
   at Microsoft.EntityFrameworkCore.DbContext.SaveChangesAsync(Boolean acceptAllChangesOnSuccess, CancellationToken cancellationToken)
   at BpmBaseApi.Persistence.Repositories.JournaledGenericRepository`1.RaiseEvent(BaseEntityEvent event, CancellationToken cancellationToken, Boolean autoCommit) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi.Persistence\Repositories\JournaledGenericRepository.cs:line 124
   at BpmBaseApi.Application.CommandHandlers.Process.SendAcquaintanceCommandHandler.Handle(SendAcquaintanceCommand cmd, CancellationToken ct) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi.Application\CommandHandlers\Process\SendAcquaintanceCommandHandler.cs:line 52
   at BpmBaseApi.Controllers.V1.ProcessController.SendAsync(SendAcquaintanceCommand cmd, CancellationToken ct) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi\Controllers\V1\ProcessController.cs:line 632
   at Microsoft.AspNetCore.Mvc.Infrastructure.ActionMethodExecutor.TaskOfIActionResultExecutor.Execute(ActionContext actionContext, IActionResultTypeMapper mapper, ObjectMethodExecutor executor, Object controller, Object[] arguments)
   at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.<InvokeActionMethodAsync>g__Logged|12_1(ControllerActionInvoker invoker)
   at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.<InvokeNextActionFilterAsync>g__Awaited|10_0(ControllerActionInvoker invoker, Task lastTask, State next, Scope scope, Object state, Boolean isCompleted)
   at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.Rethrow(ActionExecutedContextSealed context)
   at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.Next(State& next, Scope& scope, Object& state, Boolean& isCompleted)
   at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.<InvokeInnerFilterAsync>g__Awaited|13_0(ControllerActionInvoker invoker, Task lastTask, State next, Scope scope, Object state, Boolean isCompleted)
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
Accept: text/plain; x-api-version=1.0
Connection: keep-alive
Host: localhost:5143
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/140.0.0.0 Safari/537.36
Accept-Encoding: gzip, deflate, br, zstd
Accept-Language: ru-RU,ru;q=0.9,en-US;q=0.8,en;q=0.7
Content-Type: application/json; x-api-version=1.0
Origin: http://localhost:5143
Referer: http://localhost:5143/swagger/index.html
Content-Length: 470
sec-ch-ua-platform: "Windows"
sec-ch-ua: "Chromium";v="140", "Not=A?Brand";v="24", "Google Chrome";v="140"
sec-ch-ua-mobile: ?0
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Correlation-Id: 017c2cc4-7c48-4771-953e-ff9e7ba39599




namespace BpmBaseApi.Shared.Enum
{
    public enum ProcessStage
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

        /// <summary>
        /// Доработка
        /// </summary>
        Rework,

        /// <summary>
        /// Регистрация
        /// </summary>
        Registration,

        /// <summary>
        /// Перевод
        /// </summary>
        Translation,

        /// <summary>
        /// Регистратор
        /// </summary>
        Registrar,

        /// <summary>
        /// Завершено
        /// </summary>
        Completed,

        /// <summary>
        /// Заявка на изменение
        /// </summary>
        ChangeRequest,

        /// <summary>
        /// Классификация изменения и оценка срока на анализ
        /// </summary>
        ClassifyChange,

        /// <summary>
        /// Очередь на анализ
        /// </summary>
        AnalysisQueue,

        /// <summary>
        /// Анализ
        /// </summary>
        Analysis,

        /// <summary>
        /// Очередь на тестирование
        /// </summary>
        TestQueue,

        /// <summary>
        /// Тестирование
        /// </summary>
        Testing,

        /// <summary>
        /// Подготовка акта
        /// </summary>
        PrepareAct,

        /// <summary>
        /// Подписание акта
        /// </summary>
        SignAct,

        /// <summary>
        /// Оценка заказчика
        /// </summary>
        CustomerReview,

        /// <summary>
        /// Оценка срока на разработку
        /// </summary>
        DevEstimate,

        /// <summary>
        /// Очередь на разработку
        /// </summary>
        DevQueue,

        /// <summary>
        /// Разработка
        /// </summary>
        Development,

        /// <summary>
        /// Подготовка на публикацию
        /// </summary>
        PrepareRelease,

        /// <summary>
        /// Проверка исходного кода
        /// </summary>
        CodeReview,

        /// <summary>
        /// Публикация
        /// </summary>
        Release,

        /// <summary>
        /// Нормальное изменение
        /// </summary>
        NormalChange,

        /// <summary>
        /// Стандартное изменение
        /// </summary>
        StandardChange,

        /// <summary>
        /// Экстренное изменение
        /// </summary>
        EmergencyChange,

        /// <summary>
        /// Отменено
        /// </summary>
        Canceled,

        /// <summary>
        /// Отклонено
        /// </summary>
        Rejected,

        /// <summary>
        /// Утверждение
        /// </summary>
        Confirmation,

        /// <summary>
        /// Размещение
        /// </summary>
        Placements,

        Statement
    }
}
Я думаю не action А в stage  /// <summary>Отправлено на ознакомление</summary>
        Acquaintance + это и испарвить полностью  using System.Text.Json;
using System.Text.Json.Nodes;
using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Enum;
using BpmBaseApi.Shared.Models.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Application.CommandHandlers.Process;

// ====== SEND ======
public class SendAcquaintanceCommandHandler(
    IUnitOfWork unitOfWork,
    IProcessTaskService processTaskService
) : IRequestHandler<SendAcquaintanceCommand, BaseResponseDto<SendProcessResponse>>
{
    private const string AcquaintanceCode = "Acquaintance";

    public async Task<BaseResponseDto<SendProcessResponse>> Handle(SendAcquaintanceCommand cmd, CancellationToken ct)
    {
        var pd = await unitOfWork.ProcessDataRepository.GetByIdAsync(ct, cmd.ProcessDataId)
                 ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

        // Если нужно имя процесса — подтянем (опционально)
        var proc = await unitOfWork.ProcessRepository.GetByFilterAsync(ct, p => p.ProcessCode == pd.ProcessCode);

        var (recipients, comment) = TryReadFromPayload(cmd.PayloadJson);
        if (recipients.Count == 0)
            throw new HandlerException("В payload.acquaintance.recipients не найден ни один получатель",
                ErrorCodesEnum.Business);

        // Идемпотентность: не дублировать Pending по тем же пользователям
        var existingPending = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
            ct,
            t => t.ProcessDataId == pd.Id
                 && t.BlockCode == AcquaintanceCode
                 && t.Status == "Pending"
                 && !string.IsNullOrEmpty(t.AssigneeCode));

        var alreadyCodes = existingPending.Select(t => t.AssigneeCode!).ToHashSet(StringComparer.OrdinalIgnoreCase);
        var toCreate = recipients.Where(r => !alreadyCodes.Contains(r.UserCode)).ToList();
        if (toCreate.Count == 0)
            return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };

        // Родитель (скрытый)
        var parentId = Guid.NewGuid();
        await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
        {
            EntityId = parentId,
            ProcessDataId = pd.Id,
            ProcessCode = pd.ProcessCode, // <<< ОБЯЗАТЕЛЬНО
            ProcessName = proc?.ProcessName, // (опционально)
            RegNumber = pd.RegNumber, // (опционально)
            Title = pd.Title, // (опционально)
            BlockCode = AcquaintanceCode,
            BlockName = "Ознакомление",
            AssigneeCode = "system",
            Status = "Hidden",
            Comment = comment
        }, ct);

        // Дочерние Pending-задачи
        foreach (var r in toCreate)
        {
            await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
            {
                EntityId = Guid.NewGuid(),
                ParentTaskId = parentId,
                ProcessDataId = pd.Id,
                ProcessCode = pd.ProcessCode, // <<< ОБЯЗАТЕЛЬНО
                ProcessName = proc?.ProcessName, // (опционально)
                RegNumber = pd.RegNumber, // (опционально)
                Title = pd.Title, // (опционально)
                BlockCode = AcquaintanceCode,
                BlockName = "Ознакомление",
                AssigneeCode = r.UserCode,
                AssigneeName = r.UserName,
                Status = "Pending",
                Comment = comment
            }, ct);

            await processTaskService.RefreshUserTaskCacheAsync(r.UserCode, ct);
        }

        // История
        await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(new ProcessTaskHistoryCreatedEvent
        {
            ProcessDataId = pd.Id,
            TaskId = parentId,
            BlockCode = AcquaintanceCode,
            BlockName = "Ознакомление",
            Action = ProcessAction.Acquaintance.ToString(),
            ActionName = "Отправлено на ознакомление",
            Timestamp = DateTime.Now,
            Description = string.Join(", ", toCreate.Select(x => $"{x.UserName}({x.UserCode})"))
        }, ct);

        return new BaseResponseDto<SendProcessResponse>
        {
            Data = new SendProcessResponse { Success = true }
        };
    }

    private static (List<UserInfo> Recipients, string? Comment) TryReadFromPayload(Dictionary<string, object>? payload)
    {
        var recipients = new List<UserInfo>();
        string? comment = null;

        if (payload is null || payload.Count == 0) return (recipients, null);

        try
        {
            var json = JsonSerializer.Serialize(payload);
            var root = JsonNode.Parse(json)!.AsObject();
            var acq = root["acquaintance"] as JsonObject;
            if (acq is null) return (recipients, null);

            // поддержим и "comment", и "message"
            comment = acq["comment"]?.GetValue<string>()
                      ?? acq["message"]?.GetValue<string>();

            if (acq["recipients"] is JsonArray arr)
            {
                foreach (var n in arr.OfType<JsonObject>())
                {
                    var code = n["userCode"]?.GetValue<string>()?.Trim();
                    if (string.IsNullOrWhiteSpace(code)) continue;
                    var name = n["userName"]?.GetValue<string>();
                    recipients.Add(new UserInfo { UserCode = code!, UserName = name });
                }
            }
        }
        catch
        {
            /* игнорируем кривой json */
        }

        // dedupe
        recipients = recipients
            .GroupBy(x => x.UserCode, StringComparer.OrdinalIgnoreCase)
            .Select(g => g.First())
            .ToList();

        return (recipients, comment);
    }
}
using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Application.CommandHandlers.Process;

// ====== ACK ======
public class AcknowledgeAcquaintanceCommandHandler(
    IUnitOfWork unitOfWork,
    IProcessTaskService processTaskService
) : IRequestHandler<AcknowledgeAcquaintanceCommand, BaseResponseDto<SendProcessResponse>>
{
    private const string AcquaintanceCode = "Acquaintance";

    public async Task<BaseResponseDto<SendProcessResponse>> Handle(AcknowledgeAcquaintanceCommand cmd,
        CancellationToken ct)
    {
        var task = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(ct, t => t.Id == cmd.TaskId)
                   ?? throw new HandlerException("Задача не найдена", ErrorCodesEnum.Business);

        if (!string.Equals(task.BlockCode, AcquaintanceCode, StringComparison.OrdinalIgnoreCase))
            throw new HandlerException("Эта задача не является задачею ознакомления", ErrorCodesEnum.Business);

        // идемпотентность — если не Pending, просто успех
        if (!string.Equals(task.Status, "Pending", StringComparison.OrdinalIgnoreCase))
            return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };

        // История
        await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(new ProcessTaskHistoryCreatedEvent
        {
            ProcessDataId = task.ProcessDataId,
            TaskId = task.Id,
            BlockCode = task.BlockCode,
            BlockName = task.BlockName,
            Action = "Acknowledge",
            ActionName = "Ознакомлен",
            Timestamp = DateTime.Now,
            AssigneeCode = task.AssigneeCode,
            AssigneeName = task.AssigneeName,
            Description = "Пользователь подтвердил ознакомление"
        }, ct);

        // Финализируем (уберём из инбокса)
        await processTaskService.FinalizeTaskAsync(task, ct);
        if (!string.IsNullOrWhiteSpace(task.AssigneeCode))
            await processTaskService.RefreshUserTaskCacheAsync(task.AssigneeCode, ct);

        // Закрыть родителя, если больше никто не Pending
        if (task.ParentTaskId.HasValue)
        {
            var stillPending = await unitOfWork.ProcessTaskRepository.CountAsync(
                ct, t => t.ParentTaskId == task.ParentTaskId && t.Status == "Pending");

            if (stillPending == 0)
            {
                var parent = await unitOfWork.ProcessTaskRepository.GetByIdAsync(ct, task.ParentTaskId.Value);
                if (parent is not null)
                    await processTaskService.FinalizeTaskAsync(parent, ct);
            }
        }

        return new BaseResponseDto<SendProcessResponse>
        {
            Data = new SendProcessResponse { Success = true }
        };
    }
}
