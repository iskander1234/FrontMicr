на схеме ошибка они указали line2 ошибка есть  line1 Так же мне надо сделать теперь так и каждый line делать таким образом "processData": {
        "level2": {
          "formData": {
            "execution": {code:"toSecondLine", name: "На 2-линию"}  
          }
        }
      } 

      и нужно добавить Line2 ExternalOrg false Executed true 
      

Есть проблема тут Task<Guid> RaiseEvent(BaseEntityEvent @event, CancellationToken cancellationToken, bool autoCommit = true);


     // NOTE: замените на ваш реальный ивент/метод обновления payload
                    await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataPayloadChangedEvent
                    {
                        EntityId   = processData.Id,
                        PayloadJson = updatedPayload
                    }, ct);


    using System.Text.Json;
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

            // 2) читаем classification из входящего payload (как раньше)
            var incomingClassification = ReadClassification(command.PayloadJson); // может быть null

            // 2.1) если пришёл новый classification и он отличается от сохранённого в процессе — обновим payload процесса
            if (!string.IsNullOrWhiteSpace(incomingClassification))
            {
                var storedClassification = ReadClassificationFromJson(processData.PayloadJson);
                if (!string.Equals(storedClassification, incomingClassification, StringComparison.OrdinalIgnoreCase))
                {
                    var updatedPayload = UpsertClassificationIntoJson(
                        processData.PayloadJson,
                        incomingClassification!);

                    // NOTE: замените на ваш реальный ивент/метод обновления payload
                    await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataPayloadChangedEvent
                    {
                        EntityId   = processData.Id,
                        PayloadJson = updatedPayload
                    }, ct);
                }
            }

            if (string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
            {
                var claimed = await camundaService.CamundaClaimedTasks(
                                  processData.ProcessInstanceId, parentTask.BlockCode)
                              ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);

                var variables = new Dictionary<string, object>();

                // отправляем classification в Camunda только если он пришёл
                if (!string.IsNullOrWhiteSpace(incomingClassification))
                    variables["classification"] = incomingClassification;

                // ---- ДОПОЛНЕНИЕ: ветка второй схемы для Level1 ----
                if (Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var stage) && stage == ProcessStage.Level1)
                {
                    // читаем processData.level1.formData.execution ("executed"|"toSecondLine")
                    var execution = ReadLevel1Execution(command.PayloadJson);
                    bool line1 = execution switch
                    {
                        "executed"     => true,
                        "toSecondLine" => false,
                        _              => true // дефолт при отсутствии поля
                    };

                    // имя переменной должно совпадать с BPMN (на схеме "Line 1")
                    variables["Line1"] = line1;
                }
                // ---- КОНЕЦ ДОПОЛНЕНИЯ ----

                var submit = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
                {
                    TaskId    = claimed.TaskId,
                    Variables = variables
                });

                if (!submit.Success)
                    throw new HandlerException($"Ошибка при отправке Msg:{submit.Msg}", ErrorCodesEnum.Camunda);

                // сначала лог/закрыть текущую
                await processTaskService.LogHistoryItsmAsync(currentTask, command, currentTask.AssigneeCode, ct);
                await processTaskService.FinalizeTaskAsync(currentTask, ct);
                await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, ct);

                // затем закрыть родителя (system)
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

        // ====== helpers ======

        /// <summary>
        /// Старое правило: вернуть "Emergency"|"Standard"|"Normal" из входного payload (Dictionary).
        /// </summary>
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

        /// <summary>
        /// Прочитать classification из сохранённого payload процесса (JSON-строка).
        /// Использует ту же логику поиска.
        /// </summary>
        private static string? ReadClassificationFromJson(string? json)
        {
            if (string.IsNullOrWhiteSpace(json)) return null;

            try
            {
                var root = JsonNode.Parse(json) as JsonObject;
                if (root is null) return null;

                // processData.analyst.classificationCode
                var pd = root.TryGetPropertyCaseInsensitive("processData") as JsonObject;
                var analyst = pd?.TryGetPropertyCaseInsensitive("analyst") as JsonObject;
                var code = analyst?.TryGetStringCaseInsensitive("classificationCode");
                if (!string.IsNullOrWhiteSpace(code)) return Canon(code);

                // запасные ключи
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

        /// <summary>
        /// Апсертит classification в JSON-пейлоад процесса:
        /// - processData.analyst.classificationCode = {value}
        /// - при наличии плоского поля classification — синхронизирует и его.
        /// Возвращает новую JSON-строку.
        /// </summary>
        private static string UpsertClassificationIntoJson(string? json, string value)
        {
            var root = (JsonNode.Parse(string.IsNullOrWhiteSpace(json) ? "{}" : json) as JsonObject) ?? new JsonObject();

            // ensure processData.analyst exists
            var pd = root.EnsureObject("processData");
            var analyst = pd.EnsureObject("analyst");
            analyst["classificationCode"] = value;

            // sync flat "classification" if было в корне или в processData
            if (root.ContainsKeyIgnoreCase("classification"))
                root.SetStringCaseInsensitive("classification", value);

            if (pd.ContainsKeyIgnoreCase("classification"))
                pd.SetStringCaseInsensitive("classification", value);

            return root.ToJsonString(new JsonSerializerOptions { WriteIndented = false });
        }

        /// <summary>
        /// Для второй схемы (Level1): processData.level1.formData.execution
        /// </summary>
        private static string? ReadLevel1Execution(Dictionary<string, object>? payload)
        {
                 try
                 {
                     if (payload is null) return null;
            
                    if (!payload.TryGetValue("payload", out var pRaw) || pRaw is null) return null;
                     var p = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pRaw));
            
                     if (p is null || !p.TryGetValue("processData", out var pdRaw) || pdRaw is null) return null;
                     var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pdRaw));
            
                     if (pd is null || !pd.TryGetValue("level1", out var lRaw) || lRaw is null) return null;
                     var level1 = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(lRaw));
            
                     if (level1 is null || !level1.TryGetValue("formData", out var fdRaw) || fdRaw is null) return null;
                     var form = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(fdRaw));
            
                     if (form is null || !form.TryGetValue("execution", out var exRaw) || exRaw is null) return null;
                     var execution = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(exRaw));
                     
                     if (execution is null || !form.TryGetValue("code", out var codeRaw) || codeRaw is null) return null;
                     return JsonSerializer.Deserialize<string>(JsonSerializer.Serialize(codeRaw));
                 }
                 catch
                 {
                     return null;
                 }
        }
    }

    // ===== JsonNode helpers (кейсы/поиск ключей без учёта регистра) =====

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
            // если ключ с другим кейсом уже есть — перезапишем его
            foreach (var k in obj.Select(kv => kv.Key).ToList())
            {
                if (string.Equals(k, key, StringComparison.OrdinalIgnoreCase))
                {
                    obj[k] = value;
                    return;
                }
            }
            // иначе создадим новый
            obj[key] = value;
        }
    }

    // ==== заглушка события — замени названием/типом на реальный у тебя в проекте ====
    public class ProcessDataPayloadChangedEvent : BaseEntity
    {
        public Guid EntityId { get; set; }
        public string PayloadJson { get; set; } = "";
    }
}
