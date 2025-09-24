теперь мы сломали classification по смотри в чем там проблема using System.Text.Json;
using System.Text.Json.Nodes;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Models.Camunda;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.SeedWork;
using BpmBaseApi.Shared.Enum;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    /// <summary>
    /// ITSM Send:
    /// - читает classification из payloadJson (processData.analyst.classificationCode | classification | itsmLevel | level | priority)
    /// - при системном parent: отправляет в Camunda submit с переменной classification
    /// - для Level1 дополнительно шлёт булевую переменную "Line 1" по execution (executed/toSecondLine)
    /// - если classification в запросе отличается от сохранённого в процессе — обновляет payload процесса
    /// - иначе поднимает родителя в Pending
    /// </summary>
    public class SendProcessITSMCommandHandler(
        IUnitOfWork unitOfWork,
        ICamundaService camundaService,
        IProcessTaskService processTaskService)
        : IRequestHandler<SendItsmProcessCommand, BaseResponseDto<SendProcessResponse>>
    {
        public async Task<BaseResponseDto<SendProcessResponse>> Handle(
            SendItsmProcessCommand command, CancellationToken ct)
        {
            // 1) текущая подзадача должна быть Pending
            var currentTask = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                                  ct, t => t.Id == command.TaskId && t.Status == "Pending")
                              ?? throw new HandlerException("Задача не найдена или уже обработана", ErrorCodesEnum.Business);

            var processData = await unitOfWork.ProcessDataRepository.GetByIdAsync(ct, currentTask.ProcessDataId)
                              ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

            var parentTask = await unitOfWork.ProcessTaskRepository.GetByIdAsync(ct, currentTask.ParentTaskId!.Value);

            // 2) classification (как раньше) + апдейт payload при изменении
            var incomingClassification = ReadClassification(command.PayloadJson); // может быть null
            if (!string.IsNullOrWhiteSpace(incomingClassification))
            {
                var stored = ReadClassificationFromJson(processData.PayloadJson);
                if (!string.Equals(stored, incomingClassification, StringComparison.OrdinalIgnoreCase))
                {
                    var updatedPayload = UpsertClassificationIntoJson(processData.PayloadJson, incomingClassification!);

                    await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataPayloadChangedEvent
                    {
                        EntityId    = processData.Id,
                        PayloadJson = updatedPayload
                    }, ct);
                }
            }

            if (string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
            {
                var claimed = await camundaService.CamundaClaimedTasks(
                                  processData.ProcessInstanceId, parentTask.BlockCode)
                              ?? throw new HandlerException(
                                     $"Не найдена задача в Camunda: {processData.ProcessInstanceId} {parentTask.BlockCode};",
                                     ErrorCodesEnum.Camunda);

                var variables = new Dictionary<string, object>();

                // classification шлём только если пришло
                if (!string.IsNullOrWhiteSpace(incomingClassification))
                    variables["classification"] = incomingClassification;

                // ===== Вторая схема: Level1 / Level2 =====
                if (Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var stage))
                {
                    if (stage == ProcessStage.Level1)
                    {
                        var code = ReadExecutionCode(command.PayloadJson, levelIndex: 1); // "executed"|"toSecondLine"|null
                        bool line1 = code?.Equals("executed", StringComparison.OrdinalIgnoreCase) == true
                                     ? true
                                     : code?.Equals("toSecondLine", StringComparison.OrdinalIgnoreCase) == true
                                         ? false
                                         : true; // дефолт
                        variables["Line1"] = line1;
                    }
                    else if (stage == ProcessStage.Level2)
                    {
                        // ожидания: "Executed" | "ExternalOrg" | "Line3"
                        var code = ReadExecutionCode(command.PayloadJson, levelIndex: 2);

                        string? line2 = code switch
                        {
                            var c when c?.Equals("Executed",    StringComparison.OrdinalIgnoreCase) == true => "Executed",
                            var c when c?.Equals("ExternalOrg", StringComparison.OrdinalIgnoreCase) == true => "ExternalOrg",
                            // на схеме "Line1 = Line3" — для Level2 кладём явную строку "Line3"
                            var c when c?.Equals("Line3",       StringComparison.OrdinalIgnoreCase) == true
                                   || c?.Equals("toThirdLine", StringComparison.OrdinalIgnoreCase) == true   => "Line3",
                            _ => null
                        };

                        if (!string.IsNullOrWhiteSpace(line2))
                            variables["Line2"] = line2!;
                    }
                    else if (stage == ProcessStage.Level3)
                    {
                        // при необходимости можно добавить Line3 аналогично
                        // var code = ReadExecutionCode(command.PayloadJson, levelIndex: 3);
                        // variables["Line3"] = code; // если по схеме требуется строка
                    }
                }
                // ===== /Вторая схема =====

                var submit = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
                {
                    TaskId    = claimed.TaskId,
                    Variables = variables
                });

                if (!submit.Success)
                    throw new HandlerException($"Ошибка при отправке Msg:{submit.Msg}", ErrorCodesEnum.Camunda);

                // Лог + закрытие текущей (дочерней)
                await processTaskService.LogHistoryItsmAsync(currentTask, command, currentTask.AssigneeCode, ct);
                await processTaskService.FinalizeTaskAsync(currentTask, ct);
                await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, ct);

                // Финализация родителя (system)
                await processTaskService.FinalizeTaskAsync(parentTask, ct);
            }
            else
            {
                await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
                {
                    EntityId = parentTask.Id,
                    Status   = "Pending"
                }, ct);

                if (!string.IsNullOrWhiteSpace(parentTask.AssigneeCode))
                    await processTaskService.RefreshUserTaskCacheAsync(parentTask.AssigneeCode, ct);

                await processTaskService.LogHistoryItsmAsync(currentTask, command, currentTask.AssigneeCode, ct);
                await processTaskService.FinalizeTaskAsync(currentTask, ct);
                await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, ct);
            }

            return new BaseResponseDto<SendProcessResponse>
            {
                Data = new SendProcessResponse { Success = true }
            };
        }

        // ===== helpers =====

        private static string? ReadClassification(Dictionary<string, object>? payload)
        {
            if (payload is null) return null;

            static string? ReadStr(Dictionary<string, object> p, string key)
            {
                if (!p.TryGetValue(key, out var v) || v is null) return null;
                var json = JsonSerializer.Serialize(v);
                return JsonSerializer.Deserialize<string>(json, new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
            }

            string? fromAnalyst = null;
            if (payload.TryGetValue("processData", out var pdRaw) && pdRaw is not null)
            {
                var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(
                    JsonSerializer.Serialize(pdRaw), new JsonSerializerOptions { PropertyNameCaseInsensitive = true });

                if (pd != null && pd.TryGetValue("analyst", out var aRaw) && aRaw is not null)
                {
                    var analyst = JsonSerializer.Deserialize<Dictionary<string, object>>(
                        JsonSerializer.Serialize(aRaw),
                        new JsonSerializerOptions { PropertyNameCaseInsensitive = true });

                    if (analyst != null)
                        fromAnalyst = ReadStr(analyst, "classificationCode");
                }
            }

            var raw = fromAnalyst
                      ?? ReadStr(payload, "classification")
                      ?? ReadStr(payload, "itsmLevel")
                      ?? ReadStr(payload, "level")
                      ?? ReadStr(payload, "priority");

            if (string.IsNullOrWhiteSpace(raw)) return null;

            var n = raw.Trim().ToLowerInvariant();
            return n switch
            {
                "emergency" => "Emergency",
                "standard"  => "Standard",
                "normal"    => "Normal",
                _           => null
            };
        }

        private static string? ReadClassificationFromJson(string? json)
        {
            if (string.IsNullOrWhiteSpace(json)) return null;

            try
            {
                var root = JsonNode.Parse(json) as JsonObject;
                if (root is null) return null;

                var pd = root.TryGetPropertyCaseInsensitive("processData") as JsonObject;
                var analyst = pd?.TryGetPropertyCaseInsensitive("analyst") as JsonObject;
                var code = analyst?.TryGetStringCaseInsensitive("classificationCode");
                if (!string.IsNullOrWhiteSpace(code)) return Canon(code);

                var flat = root.TryGetStringCaseInsensitive("classification")
                           ?? root.TryGetStringCaseInsensitive("itsmLevel")
                           ?? root.TryGetStringCaseInsensitive("level")
                           ?? root.TryGetStringCaseInsensitive("priority");

                return Canon(flat);

                static string? Canon(string? v)
                {
                    if (string.IsNullOrWhiteSpace(v)) return null;
                    var n = v.Trim().ToLowerInvariant();
                    return n switch
                    {
                        "emergency" => "Emergency",
                        "standard"  => "Standard",
                        "normal"    => "Normal",
                        _           => null
                    };
                }
            }
            catch { return null; }
        }

        private static string UpsertClassificationIntoJson(string? json, string value)
        {
            var root = (JsonNode.Parse(string.IsNullOrWhiteSpace(json) ? "{}" : json) as JsonObject) ?? new JsonObject();

            var pd = root.EnsureObject("processData");
            var analyst = pd.EnsureObject("analyst");
            analyst["classificationCode"] = value;

            if (root.ContainsKeyIgnoreCase("classification"))
                root.SetStringCaseInsensitive("classification", value);
            if (pd.ContainsKeyIgnoreCase("classification"))
                pd.SetStringCaseInsensitive("classification", value);

            return root.ToJsonString();
        }

        /// Универсальный парсер execution.code для level1/level2/level3.
        /// Поддерживает строки и объекты { code, name }.
        private static string? ReadExecutionCode(Dictionary<string, object>? payload, int levelIndex)
        {
            try
            {
                if (payload is null) return null;

                if (!payload.TryGetValue("payload", out var pRaw) || pRaw is null) return null;
                var p = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pRaw));

                if (p is null || !p.TryGetValue("processData", out var pdRaw) || pdRaw is null) return null;
                var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pdRaw));

                var levelKey = $"level{levelIndex}";
                if (pd is null || !pd.TryGetValue(levelKey, out var lRaw) || lRaw is null) return null;
                var level = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(lRaw));

                if (level is null || !level.TryGetValue("formData", out var fdRaw) || fdRaw is null) return null;
                var form = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(fdRaw));
                if (form is null || !form.TryGetValue("execution", out var exRaw) || exRaw is null) return null;

                // 1) execution — строка?
                if (exRaw is JsonElement je && je.ValueKind == JsonValueKind.String)
                    return je.GetString();

                // 2) execution — объект {code,name}
                var exDict = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(exRaw));
                if (exDict != null && exDict.TryGetValue("code", out var codeRaw) && codeRaw != null)
                    return JsonSerializer.Deserialize<string>(JsonSerializer.Serialize(codeRaw));

                return null;
            }
            catch
            {
                return null;
            }
        }
    }

    // ==== JsonNode helpers ====
    internal static class JsonCaseHelpers
    {
        public static JsonNode? TryGetPropertyCaseInsensitive(this JsonObject obj, string key)
        {
            foreach (var kv in obj)
                if (string.Equals(kv.Key, key, StringComparison.OrdinalIgnoreCase)) return kv.Value;
            return null;
        }

        public static string? TryGetStringCaseInsensitive(this JsonObject obj, string key)
        {
            var n = obj.TryGetPropertyCaseInsensitive(key);
            return n?.GetValue<string>();
        }

        public static bool ContainsKeyIgnoreCase(this JsonObject obj, string key)
        {
            foreach (var kv in obj)
                if (string.Equals(kv.Key, key, StringComparison.OrdinalIgnoreCase)) return true;
            return false;
        }

        public static JsonObject EnsureObject(this JsonObject parent, string key)
        {
            var node = parent.TryGetPropertyCaseInsensitive(key);
            if (node is JsonObject o) return o;
            var created = new JsonObject();
            parent[key] = created;
            return created;
        }

        public static void SetStringCaseInsensitive(this JsonObject obj, string key, string value)
        {
            foreach (var k in obj.Select(kv => kv.Key).ToList())
            {
                if (string.Equals(k, key, StringComparison.OrdinalIgnoreCase))
                {
                    obj[k] = value;
                    return;
                }
            }
            obj[key] = value;
        }
    }

    // ==== событие для обновления payload ====
    public class ProcessDataPayloadChangedEvent : BaseEntityEvent
    {
        public Guid EntityId { get; set; }
        public string PayloadJson { get; set; } = "";
    }
}


Microsoft.EntityFrameworkCore.DbUpdateException: An error occurred while saving the entity changes. See the inner exception for details.
 ---> Npgsql.PostgresException (0x80004005): 23502: null value in column "ProcessCode" of relation "ProcessData" violates not-null constraint

DETAIL: Detail redacted as it may contain sensitive data. Specify 'Include Error Detail' in the connection string to include this information.
   at Npgsql.Internal.NpgsqlConnector.ReadMessageLong(Boolean async, DataRowLoadingMode dataRowLoadingMode, Boolean readingNotifications, Boolean isReadingPrependedMessage)
   at System.Runtime.CompilerServices.PoolingAsyncValueTaskMethodBuilder`1.StateMachineBox`1.System.Threading.Tasks.Sources.IValueTaskSource<TResult>.GetResult(Int16 token)
   at Npgsql.NpgsqlDataReader.NextResult(Boolean async, Boolean isConsuming, CancellationToken cancellationToken)
   at Npgsql.NpgsqlDataReader.NextResult(Boolean async, Boolean isConsuming, CancellationToken cancellationToken)
   at Npgsql.EntityFrameworkCore.PostgreSQL.Update.Internal.NpgsqlModificationCommandBatch.Consume(RelationalDataReader reader, Boolean async, CancellationToken cancellationToken)
  Exception data:
    Severity: ERROR
    SqlState: 23502
    MessageText: null value in column "ProcessCode" of relation "ProcessData" violates not-null constraint
    Detail: Detail redacted as it may contain sensitive data. Specify 'Include Error Detail' in the connection string to include this information.
    SchemaName: public
    TableName: ProcessData
    ColumnName: ProcessCode
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
   at BpmBaseApi.Persistence.Repositories.JournaledGenericRepository`1.RaiseEvent(BaseEntityEvent event, CancellationToken cancellationToken, Boolean autoCommit) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi.Persistence\Repositories\JournaledGenericRepository.cs:line 123
   at BpmBaseApi.Application.CommandHandlers.Process.SendProcessITSMCommandHandler.Handle(SendItsmProcessCommand command, CancellationToken ct) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi.Application\CommandHandlers\Process\SendProcessITSMCommandHandler.cs:line 54
   at BpmBaseApi.Controllers.V1.ITSMController.SendItsmAsync(SendItsmProcessCommand command, CancellationToken ct) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi\Controllers\V1\ITSMController.cs:line 53
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
Content-Length: 2505
sec-ch-ua-platform: "Windows"
sec-ch-ua: "Chromium";v="140", "Not=A?Brand";v="24", "Google Chrome";v="140"
sec-ch-ua-mobile: ?0
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Correlation-Id: 2a920e32-6cb9-4219-9cb0-2e7bac83adeb
