"C:\Program Files\JetBrains\JetBrains Rider 2025.1.2\plugins\dpa\DotFiles\JetBrains.DPA.Runner.exe" --handle=15840 --backend-pid=9820 --etw-collect-flags=67108622 --detach-event-name=dpa.detach.9820.15 --refresh-interval=1 -- C:/BPM/bpm/bpmbaseapi/BpmBaseApi/bin/Debug/net8.0/BpmBaseApi.exe
warn: Microsoft.EntityFrameworkCore.Model.Validation[10400]
      Sensitive data logging is enabled. Log entries and exception messages may include sensitive application data; this mode should only be enabled during development.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (84ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
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
         at Microsoft.Extensions.DependencyInjection.ServiceLookup.ServiceProviderEngineScope.GetService(Type serviceType)
         at Microsoft.Extensions.DependencyInjection.ServiceProviderServiceExtensions.GetRequiredService(IServiceProvider provider, Type serviceType)
         at Microsoft.Extensions.DependencyInjection.ServiceProviderServiceExtensions.GetServices(IServiceProvider provider, Type serviceType)
         at MicroElements.OpenApi.FluentValidation.ValidatorRegistryExtensions.GetValidators(IServiceProvider serviceProvider, Type modelType, ISchemaGenerationOptions options)+MoveNext() in /home/runner/work/MicroElements.Swashbuckle.FluentValidation/MicroElements.Swashbuckle.FluentValidation/src/MicroElements.OpenApi.FluentValidation/FluentValidation/ValidatorRegistry/ValidatorRegistryExtensions.cs:line 75
         at System.Collections.Generic.LargeArrayBuilder`1.AddRange(IEnumerable`1 items)
         at System.Collections.Generic.EnumerableHelpers.ToArray[T](IEnumerable`1 source)
         at MicroElements.Swashbuckle.FluentValidation.FluentValidationRules.<>c__DisplayClass5_0.<Apply>b__0() in /home/runner/work/MicroElements.Swashbuckle.FluentValidation/MicroElements.Swashbuckle.FluentValidation/src/MicroElements.Swashbuckle.FluentValidation/Swashbuckle/FluentValidationRules.cs:line 77
         at MicroElements.OpenApi.Core.Functional.Try[T](Func`1 func) in /home/runner/work/MicroElements.Swashbuckle.FluentValidation/MicroElements.Swashbuckle.FluentValidation/src/MicroElements.OpenApi.FluentValidation/Core/Functional.cs:line 19
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (75ms) [Parameters=[@__command_ProcessCode_0='ChangeRequest'], CommandType='Text', CommandTimeout='30']
      SELECT p."Id", p."CreateUserId", p."Created", p."LastUserId", p."ProcessCode", p."ProcessName", p."StartBlockId", p."Updated"
      FROM "Process" AS p
      WHERE p."ProcessCode" = @__command_ProcessCode_0
      LIMIT 1
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (13ms) [Parameters=[@__processCode_0='ChangeRequest', @__year_1='2025'], CommandType='Text', CommandTimeout='30']
      SELECT r."Id", r."CreateUserId", r."Created", r."LastNumber", r."LastUserId", r."ProcessCode", r."Updated", r."Year"
      FROM "RequestNumberCounter" AS r
      WHERE r."ProcessCode" = @__processCode_0 AND r."Year" = @__year_1
      LIMIT 1
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (5ms) [Parameters=[@__p_0='82e77b05-4c9a-4098-b566-d88e1e0c8823' (Nullable = true)], CommandType='Text', CommandTimeout='30']
      SELECT r."Id", r."CreateUserId", r."Created", r."LastNumber", r."LastUserId", r."ProcessCode", r."Updated", r."Year"
      FROM "RequestNumberCounter" AS r
      WHERE r."Id" = @__p_0
      LIMIT 1
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (12ms) [Parameters=[@p0='2025-09-18T16:24:25.2374820' (DbType = DateTime), @p1='82e77b05-4c9a-4098-b566-d88e1e0c8823', @p2='2025-09-22T14:53:36.8566438+05:00' (DbType = DateTime), @p3='{"EntityId":"82e77b05-4c9a-4098-b566-d88e1e0c8823"
,"UserName":"","EventDateTime":"2025-09-22T14:53:36.8566438+05:00","EventName":"RequestNumberCounterChangedEvent"}' (Nullable = false), @p4='RequestNumberCounterChangedEvent' (Nullable = false), @p5='False', @p6='' (Nullable = false), @p8='82e77b05-4c9a-4098-b566-d88e1e0c8823', @p7='24'], CommandType='Text', CommandTimeout='30']
      INSERT INTO "JournalEvents" ("EntityCreated", "EntityId", "EventDateTime", "EventJson", "EventName", "IsSentToQueue", "UserName")
      VALUES (@p0, @p1, @p2, @p3, @p4, @p5, @p6)
      RETURNING "Id";
      UPDATE "RequestNumberCounter" SET "LastNumber" = @p7
      WHERE "Id" = @p8;
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (2ms) [Parameters=[@__p_0='00000000-0000-0000-0000-000000000000' (Nullable = true)], CommandType='Text', CommandTimeout='30']
      SELECT p."Id", p."BlockCode", p."BlockName", p."CreateUserId", p."Created", p."InitiatorCode", p."InitiatorName", p."LastUserId", p."PayloadJson", p."ProcessCode", p."ProcessId", p."ProcessInstanceId", p."ProcessName", p."RegNumber", p."Started", p."StatusCode", p."StatusName", p."Title", p."Updated"
      FROM "ProcessData" AS p
      WHERE p."Id" = @__p_0
      LIMIT 1
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (2ms) [Parameters=[@p0='2025-09-22T14:53:38.3882411+05:00' (DbType = DateTime), @p1='cd1a8853-071b-489a-98a2-ba3cff56d2ec', @p2='2025-09-22T14:53:38.3187267+05:00' (DbType = DateTime), @p3='{"EntityId":"cd1a8853-071b-489a-98a2-ba3cff56
d2ec","UserName":"","EventDateTime":"2025-09-22T14:53:38.3187267+05:00","EventName":"ProcessDataCreatedEvent"}' (Nullable = false), @p4='ProcessDataCreatedEvent' (Nullable = false), @p5='False', @p6='' (Nullable = false), @p7='cd1a8853-071b-489a-98a2-ba3cff56d
2ec', @p8=NULL, @p9=NULL, @p10='' (Nullable = false), @p11='2025-09-22T14:53:38.3882411+05:00' (DbType = DateTime), @p12='m.ilespayev' (Nullable = false), @p13='Илеспаев Меииржан Анварович' (Nullable = false), @p14='' (Nullable = false), @p15='{"regData":{"use
rCode":"m.ilespayev","userName":"Илеспаев Меииржан Анварович","departmentId":"00ЗП-0013","departmentName":"Управление автоматизации бизнес-процессов","startdate":"2025-09-22T14:53:38.3178591+05:00","regnum":"CH-0024-2025"},"sysInfo":{"userCode":"m.ilespayev","
userName":"Илеспаев Меииржан Анварович","comment":"comment","action":"submit","condition":"string"},"initiator":{"id":611,"name":"Илеспаев Меииржан Анварович","position":"Начальник управления","login":"m.ilespayev","statusCode":6,"statusDescription":"Работа","
depId":"00ЗП-0013","depName":"Управление автоматизации бизнес-процессов","parentDepId":"00ЗП-0010","parentDepName":"Департамент цифровой трансформации","isFilial":false,"mail":"m.ilespayev@enpf.kz","localPhone":"0","mobilePhone":"+7(702) 171-71-14","isManager"
:true,"managerTabNumber":"4340","disabled":false,"tabNumber":"00ЗП-00240","loginAD":"m.ilespayev","shortName":"Илеспаев М.А."},"processData":{"CustomerId":"00ЗП-0010","CustomerName":"Департамент цифровой трансформации","DocumentTitle":"erwerwer","SystemId":"11
111111-0000-0000-0000-000000000002,11111111-0000-0000-0000-000000000003","SystemName":"1С Предприятие 8, модуль Бюджетирование,1С Предприятие 8, модуль Учет договоров","ProcessCategoryId":"","ProcessCategoryName":"","ProcessesId":"","ProcessesName":"","SystemO
wnerId":"00ЗП-0010","SystemOwnerName":"","ChangeReason":"fdsfsdf","PlanningInfo":"","CurrentFunctionality":"sdffd","Requirements":"32424","Impact":"geg","TestCases":"ergerger"}}' (Nullable = false), @p16='ChangeRequest' (Nullable = false), @p17='b11b435f-039b-
4d74-960e-d56c7697a447', @p18=NULL, @p19='Заявка на изменение' (Nullable = false), @p20='CH-0024-2025' (Nullable = false), @p21='2025-09-22T14:53:38.3234672+05:00' (DbType = DateTime), @p22='Started' (Nullable = false), @p23='В работе' (Nullable = false), @p24='erwerwer', @p25='0001-01-01T00:00:00.0000000' (DbType = DateTime)], CommandType='Text', CommandTimeout='30']
      INSERT INTO "JournalEvents" ("EntityCreated", "EntityId", "EventDateTime", "EventJson", "EventName", "IsSentToQueue", "UserName")
      VALUES (@p0, @p1, @p2, @p3, @p4, @p5, @p6)
      RETURNING "Id";
      INSERT INTO "ProcessData" ("Id", "BlockCode", "BlockName", "CreateUserId", "Created", "InitiatorCode", "InitiatorName", "LastUserId", "PayloadJson", "ProcessCode", "ProcessId", "ProcessInstanceId", "ProcessName", "RegNumber", "Started", "StatusCode", "StatusName", "Title", "Updated")
      VALUES (@p7, @p8, @p9, @p10, @p11, @p12, @p13, @p14, @p15, @p16, @p17, @p18, @p19, @p20, @p21, @p22, @p23, @p24, @p25);
info: System.Net.Http.HttpClient.Refit.Implementation.Generated+BpmBaseApiServicesCamundaServiceCamundaICamundaProvider, BpmBaseApi.Services, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null.LogicalHandler[100]
      Start processing HTTP request POST https://camundabpm.enpf.kz/core-service/processes/start?processId=ChangeRequest
info: System.Net.Http.HttpClient.Refit.Implementation.Generated+BpmBaseApiServicesCamundaServiceCamundaICamundaProvider, BpmBaseApi.Services, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null.ClientHandler[100]
      Sending HTTP request POST https://camundabpm.enpf.kz/core-service/processes/start?processId=ChangeRequest
info: System.Net.Http.HttpClient.Refit.Implementation.Generated+BpmBaseApiServicesCamundaServiceCamundaICamundaProvider, BpmBaseApi.Services, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null.ClientHandler[101]
      Received HTTP response headers after 730.1616ms - 500
info: System.Net.Http.HttpClient.Refit.Implementation.Generated+BpmBaseApiServicesCamundaServiceCamundaICamundaProvider, BpmBaseApi.Services, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null.LogicalHandler[101]
      End processing HTTP request after 789.1051ms - 500



      using System.Text.Encodings.Web;
using System.Text.Json;
using System.Text.Json.Nodes;
using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Responses.Process;
using BpmBaseApi.Shared.Models.Camunda;
using BpmBaseApi.Shared.Dtos;
using MediatR;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    /// <summary>
    /// Упрощённый старт под ITSM:
    /// - без поиска/переиспользования Draft/Started
    /// - не сливаем payload, не редактируем существующие записи
    /// - создаём ProcessData=Started, стартуем Camunda, апсертим files из payload, пишем Start в историю
    /// </summary>
    public class StartProcessITSMCommandHandler(
        IUnitOfWork unitOfWork,
        IProcessTaskService helperService,
        ICamundaService camundaService
    ) : IRequestHandler<StartProcessITSMCommand, BaseResponseDto<StartProcessResponse>>
    {
        public async Task<BaseResponseDto<StartProcessResponse>> Handle(
            StartProcessITSMCommand command,
            CancellationToken ct)
        {
            // 0) Процесс
            var process = await unitOfWork.ProcessRepository
                             .GetByFilterAsync(ct, p => p.ProcessCode == command.ProcessCode)
                         ?? throw new HandlerException($"Процесс с кодом {command.ProcessCode} не найден",
                             ErrorCodesEnum.Business);

            // 1) Готовим payload JSON (как есть, максимум — впишем regnum и startdate если их нет)
            var jsonOptions = new JsonSerializerOptions { Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping };
            var payloadJson = JsonSerializer.Serialize(command.Payload ?? new Dictionary<string, object>(), jsonOptions);

            // вытащим заголовок документа (если есть)
            var documentTitle = TryRead<string>(payloadJson, "processData", "documentTitle");

            // если regnum не задан — сгенерируем и впишем
            var regnum = TryRead<string>(payloadJson, "regData", "regnum");
            if (string.IsNullOrWhiteSpace(regnum))
            {
                regnum = await helperService.GenerateRequestNumberAsync(command.ProcessCode, ct);
                payloadJson = SetInPayload(payloadJson, jsonOptions, ("regData", "regnum", regnum));
            }

            // всегда проставим startdate (момент старта)
            payloadJson = SetInPayload(payloadJson, jsonOptions, ("regData", "startdate", DateTime.Now.ToString("o")));

            // 2) Создаём ProcessData = Started
            var created = new ProcessDataCreatedEvent
            {
                ProcessId     = process.Id,
                ProcessCode   = process.ProcessCode,
                ProcessName   = process.ProcessName,
                RegNumber     = regnum,
                InitiatorCode = command.InitiatorCode,
                InitiatorName = command.InitiatorName,
                StatusCode    = "Started",
                StatusName    = "В работе",
                PayloadJson   = payloadJson,
                Title         = documentTitle,
                StartedAt     = DateTime.Now
            };
            await unitOfWork.ProcessDataRepository.RaiseEvent(created, ct);

            // 3) Стартуем Camunda и связываем
            var vars = CamundaVars(
                ("processGuid",   created.EntityId.ToString(), "String"),
                ("initiatorCode", command.InitiatorCode,       "String"),
                ("initiatorName", command.InitiatorName,       "String"),
                // если есть приоритет/уровень — отдаём (англ. вариант, как договаривались)
                ("itsmLevel",     ReadItsmLevel(payloadJson),  "String") // null будет проигнорирован
            );

            var pi = await camundaService.CamundaStartProcess(new CamundaStartProcessRequest
            {
                processCode = command.ProcessCode,
                variables   = vars
            });

            await unitOfWork.ProcessDataRepository.RaiseEvent(
                new ProcessDataProcessInstanseIdChangedEvent
                {
                    EntityId          = created.EntityId,
                    ProcessInstanceId = pi
                },
                ct);

            // 4) Апсертим files из payload (если есть)
            await UpsertFilesFromPayloadAsync(created.EntityId, payloadJson, unitOfWork, ct);

            // 5) Пишем историю "Start"
            await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(new ProcessTaskHistoryCreatedEvent
            {
                ProcessDataId = created.EntityId,
                TaskId        = created.EntityId,
                Action        = "Start",
                ActionName    = "Зарегистрировано",
                BlockName     = "Регистрационная форма",
                Timestamp     = DateTime.Now,
                PayloadJson   = payloadJson,
                Comment       = "",
                Description   = "",

                // важные not null
                ProcessCode   = process.ProcessCode,
                ProcessName   = process.ProcessName,
                RegNumber     = regnum,
                InitiatorCode = command.InitiatorCode,
                InitiatorName = command.InitiatorName,
                AssigneeCode  = command.InitiatorCode,
                AssigneeName  = command.InitiatorName,
                Title         = documentTitle
            }, ct);

            return new BaseResponseDto<StartProcessResponse>
            {
                Data = new StartProcessResponse
                {
                    ProcessGuid = created.EntityId,
                    RegNumber   = regnum
                }
            };
        }

        // ===== helpers =====
        static Dictionary<string, object> CamundaVars(params (string name, object? value, string type)[] items)
        {
            var d = new Dictionary<string, object>(StringComparer.Ordinal);
            foreach (var it in items)
            {
                if (it.value is null) continue; // пропускаем пустые
                d[it.name] = new Dictionary<string, object>
                {
                    ["value"] = it.value,
                    ["type"]  = it.type
                };
            }
            return d;
        }
        
        // только английские значения, как просили: Emergency | Standard | Normal
        static string? ReadItsmLevel(string json)
        {
            var lvl = TryRead<string>(json, "processData", "itsmLevel")
                      ?? TryRead<string>(json, "processData", "level")
                      ?? TryRead<string>(json, "processData", "priority");

            if (string.IsNullOrWhiteSpace(lvl)) return null;

            var v = lvl.Trim();
            return v.Equals("Emergency", StringComparison.OrdinalIgnoreCase) ? "Emergency" :
                v.Equals("Standard",  StringComparison.OrdinalIgnoreCase) ? "Standard"  :
                v.Equals("Normal",    StringComparison.OrdinalIgnoreCase) ? "Normal"    :
                null;
        }

        private static T? TryRead<T>(string json, params string[] path)
        {
            try
            {
                JsonNode? node = JsonNode.Parse(json);
                foreach (var p in path)
                {
                    if (node is not JsonObject obj) return default;
                    node = obj.FirstOrDefault(kv => 
                        string.Equals(kv.Key, p, StringComparison.OrdinalIgnoreCase)).Value;
                    if (node is null) return default;
                }
                return node.Deserialize<T>(new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
            }
            catch { return default; }
        }


        private static string SetInPayload(string json, JsonSerializerOptions opts, (string section, string key, object value) kv)
        {
            var root = (JsonNode.Parse(json) as JsonObject) ?? new JsonObject();

            if (root[kv.section] is not JsonObject sec)
            {
                sec = new JsonObject();
                root[kv.section] = sec;
            }

            sec[kv.key] = JsonSerializer.SerializeToNode(kv.value, kv.value.GetType(), opts);
            return JsonSerializer.Serialize(root, opts);
        }

        private static async Task UpsertFilesFromPayloadAsync(
            Guid processDataId,
            string payloadJson,
            IUnitOfWork unitOfWork,
            CancellationToken ct)
        {
            try
            {
                var root  = JsonNode.Parse(payloadJson)!.AsObject();
                var files = root["files"] as JsonArray;
                if (files is null || files.Count == 0) return;

                // берём всё (и удалённые, и активные)
                var existing = await unitOfWork.ProcessFileRepository.GetByFilterListAsync(
                    ct, f => f.ProcessDataId == processDataId);

                var byFileId = existing
                    .Where(x => x.FileId != Guid.Empty)
                    .ToDictionary(x => x.FileId, x => x);

                foreach (var node in files.OfType<JsonObject>())
                {
                    var fileIdStr = node["fileId"]?.GetValue<string>();
                    var fileName  = node["fileName"]?.GetValue<string>();
                    var fileType  = node["fileType"]?.GetValue<string>();

                    if (string.IsNullOrWhiteSpace(fileName) || string.IsNullOrWhiteSpace(fileType))
                        throw new HandlerException("Каждый объект в \"files\" должен иметь непустые fileName и fileType",
                            ErrorCodesEnum.Business);

                    if (string.IsNullOrWhiteSpace(fileIdStr) || !Guid.TryParse(fileIdStr, out var fileId))
                        throw new HandlerException("Каждый объект в \"files\" должен иметь корректный GUID в поле fileId",
                            ErrorCodesEnum.Business);

                    if (!byFileId.TryGetValue(fileId, out var ex))
                    {
                        await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileCreatedEvent
                        {
                            EntityId      = Guid.NewGuid(),
                            ProcessDataId = processDataId,
                            FileId        = fileId,
                            FileName      = fileName!,
                            FileType      = fileType!
                        }, ct);
                    }
                    else
                    {
                        if (ex.State != ProcessFileState.Active || ex.FileName != fileName || ex.FileType != fileType)
                        {
                            await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileStatusChangedEvent
                            {
                                EntityId = ex.Id,
                                State    = ProcessFileState.Active,
                                FileName = fileName,
                                FileType = fileType
                            }, ct);
                        }
                    }
                }
            }
            catch
            {
                // опционально: логирование
            }
        }
    }
}





























