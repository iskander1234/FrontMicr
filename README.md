"C:\Program Files\JetBrains\JetBrains Rider 2025.1.2\plugins\dpa\DotFiles\JetBrains.DPA.Runner.exe" --handle=3624 --backend-pid=9820 --etw-collect-flags=67108622 --detach-event-name=dpa.detach.9820.21 --refresh-interval=1 -- C:/BPM/bpm/bpmbaseapi/BpmBaseApi/bin/Debug/net8.0/BpmBaseApi.exe
warn: Microsoft.EntityFrameworkCore.Model.Validation[10400]
      Sensitive data logging is enabled. Log entries and exception messages may include sensitive application data; this mode should only be enabled during development.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (85ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT "MigrationId", "ProductVersion"
      FROM "__EFMigrationsHistory"
      ORDER BY "MigrationId";
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://localhost:5143
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Development
info: Microsoft.Hosting.Lifetime[0]
      Content root path: C:\BPM\bpm\bpmbaseapi\BpmBaseApi
warn: MicroElements.Swashbuckle.FluentValidation.FluentValidationRules[0]
      GetValidators for type 'T' failed
      System.NotSupportedException: Cannot create arrays of open type.
         at System.Array.InternalCreate(RuntimeType elementType, Int32 rank, Int32* pLengths, Int32* pLowerBounds)
         at System.Array.CreateInstance(Type elementType, Int32 length)
         at Microsoft.Extensions.DependencyInjection.ServiceLookup.CallSiteRuntimeResolver.<VisitIEnumerable>g__CreateArray|12_0(Type elementType, Int32 length)
         at Microsoft.Extensions.DependencyInjection.ServiceLookup.CallSiteRuntimeResolver.VisitIEnumerable(IEnumerableCallSite enumerableCallSite, RuntimeResolverContext context)
         at Microsoft.Extensions.DependencyInjection.ServiceLookup.CallSiteVisitor`2.VisitCallSiteMain(ServiceCallSite callSite, TArgument argument)
         at Microsoft.Extensions.DependencyInjection.ServiceLookup.CallSiteRuntimeResolver.VisitRootCache(ServiceCallSite callSite, RuntimeResolverContext context)
         at Microsoft.Extensions.DependencyInjection.ServiceLookup.CallSiteVisitor`2.VisitCallSite(ServiceCallSite callSite, TArgument argument)
         at Microsoft.Extensions.DependencyInjection.ServiceLookup.CallSiteRuntimeResolver.Resolve(ServiceCallSite callSite, ServiceProviderEngineScope scope)
         at Microsoft.Extensions.DependencyInjection.ServiceProvider.CreateServiceAccessor(ServiceIdentifier serviceIdentifier)
         at System.Collections.Concurrent.ConcurrentDictionary`2.GetOrAdd(TKey key, Func`2 valueFactory)
         at Microsoft.Extensions.DependencyInjection.ServiceProvider.GetService(ServiceIdentifier serviceIdentifier, ServiceProviderEngineScope serviceProviderEngineScope)
         at Microsoft.Extensions.DependencyInjection.ServiceLookup.ServiceProviderEngineScope.GetService(Type serviceType)
         at Microsoft.Extensions.DependencyInjection.ServiceProviderServiceExtensions.GetRequiredService(IServiceProvider provider, Type serviceType)
         at Microsoft.Extensions.DependencyInjection.ServiceProviderServiceExtensions.GetServices(IServiceProvider provider, Type serviceType)
         at MicroElements.OpenApi.FluentValidation.ValidatorRegistryExtensions.GetValidators(IServiceProvider serviceProvider, Type modelType, ISchemaGenerationOptions options)+MoveNext() in /home/runner/work/MicroElements.Swashbuckle.FluentValidation/MicroElements.Swashbuckle.FluentValidation/src/MicroElements.OpenApi.FluentValidation/FluentValidation/ValidatorRegistry/ValidatorRegistryExtensions.cs:line 75
         at System.Collections.Generic.LargeArrayBuilder`1.AddRange(IEnumerable`1 items)
         at System.Collections.Generic.EnumerableHelpers.ToArray[T](IEnumerable`1 source)
         at MicroElements.Swashbuckle.FluentValidation.FluentValidationRules.<>c__DisplayClass5_0.<Apply>b__0() in /home/runner/work/MicroElements.Swashbuckle.FluentValidation/MicroElements.Swashbuckle.FluentValidation/src/MicroElements.Swashbuckle.FluentValidation/Swashbuckle/FluentValidationRules.cs:line 77
         at MicroElements.OpenApi.Core.Functional.Try[T](Func`1 func) in /home/runner/work/MicroElements.Swashbuckle.FluentValidation/MicroElements.Swashbuckle.FluentValidation/src/MicroElements.OpenApi.FluentValidation/Core/Functional.cs:line 19
warn: MicroElements.Swashbuckle.FluentValidation.FluentValidationRules[0]
      GetValidators for type 'BpmBaseApi.Shared.Dtos.BaseResponseDto`1[T]' failed
      System.NotSupportedException: Cannot create arrays of open type.
         at System.Array.InternalCreate(RuntimeType elementType, Int32 rank, Int32* pLengths, Int32* pLowerBounds)
         at System.Array.CreateInstance(Type elementType, Int32 length)
         at Microsoft.Extensions.DependencyInjection.ServiceLookup.CallSiteRuntimeResolver.<VisitIEnumerable>g__CreateArray|12_0(Type elementType, Int32 length)
         at Microsoft.Extensions.DependencyInjection.ServiceLookup.CallSiteRuntimeResolver.VisitIEnumerable(IEnumerableCallSite enumerableCallSite, RuntimeResolverContext context)
         at Microsoft.Extensions.DependencyInjection.ServiceLookup.CallSiteVisitor`2.VisitCallSiteMain(ServiceCallSite callSite, TArgument argument)
         at Microsoft.Extensions.DependencyInjection.ServiceLookup.CallSiteRuntimeResolver.VisitRootCache(ServiceCallSite callSite, RuntimeResolverContext context)
         at Microsoft.Extensions.DependencyInjection.ServiceLookup.CallSiteVisitor`2.VisitCallSite(ServiceCallSite callSite, TArgument argument)
         at Microsoft.Extensions.DependencyInjection.ServiceLookup.CallSiteRuntimeResolver.Resolve(ServiceCallSite callSite, ServiceProviderEngineScope scope)
         at Microsoft.Extensions.DependencyInjection.ServiceProvider.CreateServiceAccessor(ServiceIdentifier serviceIdentifier)
         at System.Collections.Concurrent.ConcurrentDictionary`2.GetOrAdd(TKey key, Func`2 valueFactory)
         at Microsoft.Extensions.DependencyInjection.ServiceProvider.GetService(ServiceIdentifier serviceIdentifier, ServiceProviderEngineScope serviceProviderEngineScope)
         at Microsoft.Extensions.DependencyInjection.ServiceLookup.ServiceProviderEngineScope.GetService(Type serviceType)
         at Microsoft.Extensions.DependencyInjection.ServiceProviderServiceExtensions.GetRequiredService(IServiceProvider provider, Type serviceType)
         at Microsoft.Extensions.DependencyInjection.ServiceProviderServiceExtensions.GetServices(IServiceProvider provider, Type serviceType)
         at MicroElements.OpenApi.FluentValidation.ValidatorRegistryExtensions.GetValidators(IServiceProvider serviceProvider, Type modelType, ISchemaGenerationOptions options)+MoveNext() in /home/runner/work/MicroElements.Swashbuckle.FluentValidation/MicroElements.Swashbuckle.FluentValidation/src/MicroElements.OpenApi.FluentValidation/FluentValidation/ValidatorRegistry/ValidatorRegistryExtensions.cs:line 75
         at System.Collections.Generic.LargeArrayBuilder`1.AddRange(IEnumerable`1 items)
         at System.Collections.Generic.EnumerableHelpers.ToArray[T](IEnumerable`1 source)
         at MicroElements.Swashbuckle.FluentValidation.FluentValidationRules.<>c__DisplayClass5_0.<Apply>b__0() in /home/runner/work/MicroElements.Swashbuckle.FluentValidation/MicroElements.Swashbuckle.FluentValidation/src/MicroElements.Swashbuckle.FluentValidation/Swashbuckle/FluentValidationRules.cs:line 77
         at MicroElements.OpenApi.Core.Functional.Try[T](Func`1 func) in /home/runner/work/MicroElements.Swashbuckle.FluentValidation/MicroElements.Swashbuckle.FluentValidation/src/MicroElements.OpenApi.FluentValidation/Core/Functional.cs:line 19
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (76ms) [Parameters=[@__command_TaskId_0='3d3d1089-72ef-45d4-91fd-5e31adea23b4'], CommandType='Text', CommandTimeout='30']
      SELECT p."Id", p."AssigneeCode", p."AssigneeName", p."BlockCode", p."BlockId", p."BlockName", p."Comment", p."CreateUserId", p."Created", p."InitiatorCode", p."InitiatorName", p."LastUserId", p."Order", p."ParentTaskId", p."PayloadJson", p."ProcessCode",
 p."ProcessDataId", p."ProcessName", p."RegNumber", p."Status", p."Title", p."Updated", p0."Id", p0."BlockCode", p0."BlockName", p0."CreateUserId", p0."Created", p0."InitiatorCode", p0."InitiatorName", p0."LastUserId", p0."PayloadJson", p0."ProcessCode", p0."ProcessId", p0."ProcessInstanceId", p0."ProcessName", p0."RegNumber", p0."Started", p0."StatusCode", p0."StatusName", p0."Title", p0."Updated"
      FROM "ProcessTask" AS p
      INNER JOIN "ProcessData" AS p0 ON p."ProcessDataId" = p0."Id"
      WHERE p."Id" = @__command_TaskId_0
      LIMIT 1
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (7ms) [Parameters=[@__p_0='ed662da9-9730-4797-9d2f-b0b0b1101039' (Nullable = true)], CommandType='Text', CommandTimeout='30']
      SELECT p."Id", p."BlockCode", p."BlockName", p."CreateUserId", p."Created", p."InitiatorCode", p."InitiatorName", p."LastUserId", p."PayloadJson", p."ProcessCode", p."ProcessId", p."ProcessInstanceId", p."ProcessName", p."RegNumber", p."Started", p."StatusCode", p."StatusName", p."Title", p."Updated"
      FROM "ProcessData" AS p
      WHERE p."Id" = @__p_0
      LIMIT 1
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (3ms) [Parameters=[@__currentTask_ParentTaskId_0='a2724acd-1e52-4447-8b01-76b9f043824d' (Nullable = true), @__currentTask_Id_1='3d3d1089-72ef-45d4-91fd-5e31adea23b4'], CommandType='Text', CommandTimeout='30']
      SELECT count(*)::int
      FROM "ProcessTask" AS p
      WHERE p."ParentTaskId" = @__currentTask_ParentTaskId_0 AND p."Id" <> @__currentTask_Id_1 AND p."Status" = 'Pending'
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (3ms) [Parameters=[@__p_0='a2724acd-1e52-4447-8b01-76b9f043824d' (Nullable = true)], CommandType='Text', CommandTimeout='30']
      SELECT p."Id", p."AssigneeCode", p."AssigneeName", p."BlockCode", p."BlockId", p."BlockName", p."Comment", p."CreateUserId", p."Created", p."InitiatorCode", p."InitiatorName", p."LastUserId", p."Order", p."ParentTaskId", p."PayloadJson", p."ProcessCode",
 p."ProcessDataId", p."ProcessName", p."RegNumber", p."Status", p."Title", p."Updated", p0."Id", p0."BlockCode", p0."BlockName", p0."CreateUserId", p0."Created", p0."InitiatorCode", p0."InitiatorName", p0."LastUserId", p0."PayloadJson", p0."ProcessCode", p0."ProcessId", p0."ProcessInstanceId", p0."ProcessName", p0."RegNumber", p0."Started", p0."StatusCode", p0."StatusName", p0."Title", p0."Updated"
      FROM "ProcessTask" AS p
      INNER JOIN "ProcessData" AS p0 ON p."ProcessDataId" = p0."Id"
      WHERE p."Id" = @__p_0
      LIMIT 1
info: System.Net.Http.HttpClient.Refit.Implementation.Generated+BpmBaseApiServicesCamundaServiceCamundaICamundaProvider, BpmBaseApi.Services, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null.LogicalHandler[100]
      Start processing HTTP request GET https://camundabpm.enpf.kz/core-service/task/claimed-tasks/572631c1-97b9-11f0-857a-0242ac160004/ClassifyChange
info: System.Net.Http.HttpClient.Refit.Implementation.Generated+BpmBaseApiServicesCamundaServiceCamundaICamundaProvider, BpmBaseApi.Services, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null.ClientHandler[100]
      Sending HTTP request GET https://camundabpm.enpf.kz/core-service/task/claimed-tasks/572631c1-97b9-11f0-857a-0242ac160004/ClassifyChange
info: System.Net.Http.HttpClient.Refit.Implementation.Generated+BpmBaseApiServicesCamundaServiceCamundaICamundaProvider, BpmBaseApi.Services, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null.ClientHandler[101]
      Received HTTP response headers after 357.6965ms - 200
info: System.Net.Http.HttpClient.Refit.Implementation.Generated+BpmBaseApiServicesCamundaServiceCamundaICamundaProvider, BpmBaseApi.Services, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null.LogicalHandler[101]
      End processing HTTP request after 415.2862ms - 200
info: System.Net.Http.HttpClient.Refit.Implementation.Generated+BpmBaseApiServicesCamundaServiceCamundaICamundaProvider, BpmBaseApi.Services, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null.LogicalHandler[100]
      Start processing HTTP request POST https://camundabpm.enpf.kz/core-service/task/submit?taskId=5742ba7f-97b9-11f0-857a-0242ac160004
info: System.Net.Http.HttpClient.Refit.Implementation.Generated+BpmBaseApiServicesCamundaServiceCamundaICamundaProvider, BpmBaseApi.Services, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null.ClientHandler[100]
      Sending HTTP request POST https://camundabpm.enpf.kz/core-service/task/submit?taskId=5742ba7f-97b9-11f0-857a-0242ac160004
info: System.Net.Http.HttpClient.Refit.Implementation.Generated+BpmBaseApiServicesCamundaServiceCamundaICamundaProvider, BpmBaseApi.Services, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null.ClientHandler[101]
      Received HTTP response headers after 109.82ms - 200
info: System.Net.Http.HttpClient.Refit.Implementation.Generated+BpmBaseApiServicesCamundaServiceCamundaICamundaProvider, BpmBaseApi.Services, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null.LogicalHandler[101]
      End processing HTTP request after 110.155ms - 200


Я взял с ProcessTaskId но все равно 

{
  "data": null,
  "message": "Ошибка при отправке Msg:Error submitting task: Unknown property used in expression: ${classification == 'Normal'}. Cause: Cannot resolve identifier 'classification'",
  "errorCode": 1002
}








using System.Text.Json;
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

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    /// <summary>
    /// ITSM Send:
    /// - читает classification из payloadJson (processData.analyst.classificationCode | classification | level | priority)
    /// - всегда устанавливает переменную на инстансе процесса
    /// - далее как в обычном Send: если родитель system — submit, иначе поднимаем его в Pending
    /// </summary>
    public class SendProcessITSMCommandHandler(
        IUnitOfWork unitOfWork,
        ICamundaService camundaService,
        IProcessTaskService processTaskService)
        : IRequestHandler<SendProcessCommand, BaseResponseDto<SendProcessResponse>>
    {
        public async Task<BaseResponseDto<SendProcessResponse>> Handle(
            SendProcessCommand command, CancellationToken ct)
        {
            var currentTask = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                                  ct, t => t.Id == command.TaskId && t.Status == "Pending")
                              ?? throw new HandlerException("Задача не найдена или уже обработана",
                                  ErrorCodesEnum.Business);

            var processData = await unitOfWork.ProcessDataRepository.GetByIdAsync(ct, currentTask.ProcessDataId)
                              ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

            var parentTask = await unitOfWork.ProcessTaskRepository.GetByIdAsync(ct, currentTask.ParentTaskId!.Value);

            // Собираем classification (Emergency|Standard|Normal) из payloadJson
            var variables = BuildClassificationVariables(command.PayloadJson); // вернёт {} если не нашли

            if (string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
            {
                // Шлём переменные ВМЕСТЕ с submit — шлюзы увидят classification
                var claimed = await camundaService.CamundaClaimedTasks(
                                  processData.ProcessInstanceId, parentTask.BlockCode)
                              ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);

                var submit = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
                {
                    TaskId = claimed.TaskId,
                    Variables = variables // <= ВАЖНО
                });

                if (!submit.Success)
                    throw new HandlerException($"Ошибка при отправке Msg:{submit.Msg}", ErrorCodesEnum.Camunda);

                await processTaskService.FinalizeTaskAsync(parentTask, ct);
            }
            else
            {
                // Родитель не system: просто поднимаем его
                await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
                {
                    EntityId = parentTask.Id,
                    Status = "Pending"
                }, ct);

                if (!string.IsNullOrWhiteSpace(parentTask.AssigneeCode))
                    await processTaskService.RefreshUserTaskCacheAsync(parentTask.AssigneeCode, ct);
            }

            await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, ct);
            await processTaskService.FinalizeTaskAsync(currentTask, ct);
            await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, ct);

            return new BaseResponseDto<SendProcessResponse>
            {
                Data = new SendProcessResponse { Success = true }
            };
        }

        private static Dictionary<string, object> BuildClassificationVariables(Dictionary<string, object>? payload)
        {
            if (payload is null) return new();

            static string? ReadStr(Dictionary<string, object> p, string key)
            {
                if (!p.TryGetValue(key, out var v) || v is null) return null;
                var json = JsonSerializer.Serialize(v);
                return JsonSerializer.Deserialize<string>(json,
                    new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
            }

            // Ищем processData.analyst.classificationCode
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
                    if (analyst != null) fromAnalyst = ReadStr(analyst, "classificationCode");
                }
            }

            var level = fromAnalyst
                        ?? ReadStr(payload, "classification")
                        ?? ReadStr(payload, "itsmLevel")
                        ?? ReadStr(payload, "level")
                        ?? ReadStr(payload, "priority");

            if (string.IsNullOrWhiteSpace(level)) return new();

            var canonical = level.Trim().ToLowerInvariant() switch
            {
                "emergency" => "Emergency",
                "standard" => "Standard",
                "normal" => "Normal",
                _ => null
            };

            return canonical is null ? new() : new() { ["classification"] = canonical };
        }
    }
}














