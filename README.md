нужно добавить это чтобы var jsonOptions = new JsonSerializerOptions { Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping };
            var payloadJson = JsonSerializer.Serialize(command.Payload, jsonOptions);

в базу не попадало так 
{"regData":{"userCode":"m.ilespayev","userName":"\u0418\u043B\u0435\u0441\u043F\u0430\u0435\u0432 \u041C\u0435\u0438\u0438\u0440\u0436\u0430\u043D \u0410\u043D\u0432\u0430\u0440\u043E\u0432\u0438\u0447","departmentId":"00\u0417\u041F-0013","departmentName":"\u0423\u043F\u0440\u0430\u0432\u043B\u0435\u043D\u0438\u0435 \u0430\u0432\u0442\u043E\u043C\u0430\u0442\u0438\u0437\u0430\u0446\u0438\u0438 \u0431\u0438\u0437\u043D\u0435\u0441-\u043F\u0440\u043E\u0446\u0435\u0441\u0441\u043E\u0432","startdate":"2025-09-24T16:17:42.2547600\u002B05:00","regnum":"CH-0081-2025"},"sysInfo":{"userCode":"m.ilespayev","userName":"\u0418\u043B\u0435\u0441\u043F\u0430\u0435\u0432 \u041C\u0435\u0438\u0438\u0440\u0436\u0430\u043D \u0410\u043D\u0432\u0430\u0440\u043E\u0432\u0438\u0447","comment":"comment","action":"submit","condition":"string"},"initiator":{"id":611,"name":"\u0418\u043B\u0435\u0441\u043F\u0430\u0435\u0432 \u041C\u0435\u0438\u0438\u0440\u0436\u0430\u043D \u0410\u043D\u0432\u0430\u0440\u043E\u0432\u0438\u0447","position":"\u041D\u0430\u0447\u0430\u043B\u044C\u043D\u0438\u043A \u0443\u043F\u0440\u0430\u0432\u043B\u0435\u043D\u0438\u044F","login":"m.ilespayev","statusCode":6,"statusDescription":"\u0420\u0430\u0431\u043E\u0442\u0430","depId":"00\u0417\u041F-0013","depName":"\u0423\u043F\u0440\u0430\u0432\u043B\u0435\u043D\u0438\u0435 \u0430\u0432\u0442\u043E\u043C\u0430\u0442\u0438\u0437\u0430\u0446\u0438\u0438 \u0431\u0438\u0437\u043D\u0435\u0441-\u043F\u0440\u043E\u0446\u0435\u0441\u0441\u043E\u0432","parentDepId":"00\u0417\u041F-0010","parentDepName":"\u0414\u0435\u043F\u0430\u0440\u0442\u0430\u043C\u0435\u043D\u0442 \u0446\u0438\u0444\u0440\u043E\u0432\u043E\u0439 \u0442\u0440\u0430\u043D\u0441\u0444\u043E\u0440\u043C\u0430\u0446\u0438\u0438","isFilial":false,"mail":"m.ilespayev@enpf.kz","localPhone":"0","mobilePhone":"\u002B7(702) 171-71-14","isManager":true,"managerTabNumber":"4340","disabled":false,"tabNumber":"00\u0417\u041F-00240","loginAD":"m.ilespayev","shortName":"\u0418\u043B\u0435\u0441\u043F\u0430\u0435\u0432 \u041C.\u0410."},"processData":{"CustomerId":"00\u0417\u041F-0010","CustomerName":"\u0414\u0435\u043F\u0430\u0440\u0442\u0430\u043C\u0435\u043D\u0442 \u0446\u0438\u0444\u0440\u043E\u0432\u043E\u0439 \u0442\u0440\u0430\u043D\u0441\u0444\u043E\u0440\u043C\u0430\u0446\u0438\u0438","DocumentTitle":"erwerwer","SystemId":"11111111-0000-0000-0000-000000000002,11111111-0000-0000-0000-000000000003","SystemName":"1\u0421 \u041F\u0440\u0435\u0434\u043F\u0440\u0438\u044F\u0442\u0438\u0435 8, \u043C\u043E\u0434\u0443\u043B\u044C \u0411\u044E\u0434\u0436\u0435\u0442\u0438\u0440\u043E\u0432\u0430\u043D\u0438\u0435,1\u0421 \u041F\u0440\u0435\u0434\u043F\u0440\u0438\u044F\u0442\u0438\u0435 8, \u043C\u043E\u0434\u0443\u043B\u044C \u0423\u0447\u0435\u0442 \u0434\u043E\u0433\u043E\u0432\u043E\u0440\u043E\u0432","ProcessCategoryId":"","ProcessCategoryName":"","ProcessesId":"","ProcessesName":"","SystemOwnerId":"00\u0417\u041F-0010","SystemOwnerName":"","ChangeReason":"fdsfsdf","PlanningInfo":"","CurrentFunctionality":"sdffd","Requirements":"32424","Impact":"geg","TestCases":"ergerger","analyst":{"classificationCode":"Normal","classificationName":"\u041D\u043E\u0440\u043C\u0430\u043B\u044C\u043D\u043E\u0435 \u0438\u0437\u043C\u0435\u043D\u0435\u043D\u0438\u0435 ","deadline":"7"}}}


Вот весь

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
using BpmBaseApi.Shared.Enum;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    /// <summary>
    /// ITSM Send:
    /// - если во входном payload есть processData и он отличается — обновляем payload процесса (DB) без оглядки на classification
    /// - для схемы Level1/2/3: отправляем в Camunda соответствующие переменные (Line1/Line2/…)
    /// - иначе поднимаем родителя в Pending
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
            // 1) Текущая подзадача должна быть Pending
            var currentTask = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                                  ct, t => t.Id == command.TaskId && t.Status == "Pending")
                              ?? throw new HandlerException("Задача не найдена или уже обработана",
                                  ErrorCodesEnum.Business);

            var processData = await unitOfWork.ProcessDataRepository.GetByIdAsync(ct, currentTask.ProcessDataId)
                              ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

            var parentTask = await unitOfWork.ProcessTaskRepository.GetByIdAsync(ct, currentTask.ParentTaskId!.Value);

            // 2) ОБНОВЛЯЕМ ТОЛЬКО processData, если пришли изменения (НЕ завязано на classification)
            await UpdateProcessDataIfChangedAsync(processData, command.PayloadJson, unitOfWork, ct);

            // 3) Camunda / Pending
            if (string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
            {
                var claimed = await camundaService.CamundaClaimedTasks(
                                  processData.ProcessInstanceId, parentTask.BlockCode)
                              ?? throw new HandlerException(
                                  $"Не найдена задача в Camunda: {processData.ProcessInstanceId} {parentTask.BlockCode};",
                                  ErrorCodesEnum.Camunda);

                var variables = new Dictionary<string, object>();

                var classification = ReadClassificationLoose(command.PayloadJson, processData.PayloadJson);
                if (!string.IsNullOrWhiteSpace(classification))
                    variables["classification"] = classification;

                // ===== Схема Level1/2/3 =====
                if (Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var stage))
                {
                    if (stage == ProcessStage.Level1)
                    {
                        // "executed" => true, "toSecondLine" => false, дефолт true
                        var code = ReadExecutionCode(command.PayloadJson, levelIndex: 1);
                        bool line1 = code?.Equals("executed", StringComparison.OrdinalIgnoreCase) == true
                            ? true
                            : code?.Equals("toSecondLine", StringComparison.OrdinalIgnoreCase) == true
                                ? false
                                : true;
                        variables["Line1"] = line1; // если в BPMN имя с пробелом — замени ключ на "Line 1"
                    }
                    else if (stage == ProcessStage.Level2)
                    {
                        // строковая переменная: Executed | ExternalOrg | Line3 (toThirdLine -> Line3)
                        var code = ReadExecutionCode(command.PayloadJson, levelIndex: 2);
                        string? line2 = code switch
                        {
                            var c when c?.Equals("Executed", StringComparison.OrdinalIgnoreCase) == true => "Executed",
                            var c when c?.Equals("ExternalOrg", StringComparison.OrdinalIgnoreCase) == true =>
                                "ExternalOrg",
                            var c when c?.Equals("Line3", StringComparison.OrdinalIgnoreCase) == true
                                       || c?.Equals("toThirdLine", StringComparison.OrdinalIgnoreCase) ==
                                       true => "Line3",
                            _ => null
                        };
                        if (!string.IsNullOrWhiteSpace(line2))
                            variables["Line2"] = line2!;
                    }
                    else if (stage == ProcessStage.Level3)
                    {
                        // при необходимости — аналогично добавить Line3
                        // var code = ReadExecutionCode(command.PayloadJson, 3);
                        // variables["Line3"] = code;
                    }
                }
                // ===== /схема =====

                var submit = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
                {
                    TaskId = claimed.TaskId,
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
                    Status = "Pending"
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

        // ====================== UPDATE processData ======================

        private static async Task UpdateProcessDataIfChangedAsync(
            ProcessDataEntity processData,
            Dictionary<string, object>? incomingPayload,
            IUnitOfWork unitOfWork,
            CancellationToken ct)
        {
            var incomingProcessData = ExtractIncomingProcessData(incomingPayload);
            if (incomingProcessData is null) return; // не прислали processData — ничего не делаем

            var merged = MergeProcessDataSection(processData.PayloadJson, incomingProcessData);

            if (!string.Equals(merged, processData.PayloadJson, StringComparison.Ordinal))
            {
                // Поднимаем штатное событие редактирования (оно не трогает ProcessCode и пр. not-null поля)
                await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataEditedEvent
                {
                    EntityId = processData.Id,
                    StatusCode = processData.StatusCode,
                    StatusName = processData.StatusName,
                    PayloadJson = merged,
                    InitiatorCode = processData.InitiatorCode,
                    InitiatorName = processData.InitiatorName,
                    Title = processData.Title
                }, ct);

                // Если у тебя файлы лежат в payload — держим в синхроне
                await ReconcileFilesAsync(processData.Id, merged, unitOfWork, ct);

                // Локально обновим, чтобы ниже использовать уже новую версию
                processData.PayloadJson = merged;
            }
        }

        /// Вытаскиваем JsonObject processData из входящего payload (поддержка обоих форматов).
        private static JsonObject? ExtractIncomingProcessData(Dictionary<string, object>? payload)
        {
            if (payload is null) return null;

            // формат 1: { payload: { processData: {...} } }
            if (payload.TryGetValue("payload", out var pRaw) && pRaw is not null)
            {
                var p = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pRaw));
                if (p != null && p.TryGetValue("processData", out var pdRaw) && pdRaw is not null)
                    return JsonNode.Parse(JsonSerializer.Serialize(pdRaw)) as JsonObject;
            }

            // формат 2: { processData: {...} }
            if (payload.TryGetValue("processData", out var pdRaw2) && pdRaw2 is not null)
                return JsonNode.Parse(JsonSerializer.Serialize(pdRaw2)) as JsonObject;

            return null;
        }

        /// Глубокий merge ТОЛЬКО секции processData (массивы/примитивы — overwrite).
        private static string MergeProcessDataSection(string? baseJson, JsonObject incomingProcessData)
        {
            var root = (JsonNode.Parse(string.IsNullOrWhiteSpace(baseJson) ? "{}" : baseJson) as JsonObject) ??
                       new JsonObject();
            var pd = root["processData"] as JsonObject ?? new JsonObject();
            root["processData"] = pd;

            DeepMerge(pd, incomingProcessData);
            return root.ToJsonString(new JsonSerializerOptions { WriteIndented = false });

            static void DeepMerge(JsonObject target, JsonObject patch)
            {
                foreach (var kv in patch)
                {
                    if (kv.Value is null)
                    {
                        target[kv.Key] = null;
                        continue;
                    }

                    if (kv.Value is JsonObject patchObj)
                    {
                        if (target[kv.Key] is JsonObject targetObj)
                            DeepMerge(targetObj, patchObj);
                        else
                            target[kv.Key] = patchObj.DeepClone();
                    }
                    else
                    {
                        target[kv.Key] = kv.Value.DeepClone();
                    }
                }
            }
        }

        /// Универсальный парсер execution.code для level1/2/3 (строка или {code,name}).
        private static string? ReadExecutionCode(Dictionary<string, object>? payload, int levelIndex)
        {
            try
            {
                if (payload is null) return null;

                // поддержка обоих форматов: payload.processData.* ИЛИ processData.* на верхнем уровне
                object? pdRaw = null;

                if (payload.TryGetValue("payload", out var pRaw) && pRaw is not null)
                {
                    var p = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pRaw));
                    if (p is not null && p.TryGetValue("processData", out var pdRaw1) && pdRaw1 is not null)
                        pdRaw = pdRaw1;
                }
                else if (payload.TryGetValue("processData", out var pdRaw2) && pdRaw2 is not null)
                {
                    pdRaw = pdRaw2;
                }

                if (pdRaw is null) return null;

                var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pdRaw));
                var levelKey = $"level{levelIndex}";
                if (pd is null || !pd.TryGetValue(levelKey, out var lRaw) || lRaw is null) return null;

                var level = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(lRaw));
                if (level is null || !level.TryGetValue("formData", out var fdRaw) || fdRaw is null) return null;

                var form = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(fdRaw));
                if (form is null || !form.TryGetValue("execution", out var exRaw) || exRaw is null) return null;

                if (exRaw is JsonElement je && je.ValueKind == JsonValueKind.String)
                    return je.GetString();

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

        // ===== при необходимости — синхронизация файлов как в твоём рабочем коде =====
        private static async Task ReconcileFilesAsync(
            Guid processDataId,
            string payloadJson,
            IUnitOfWork unitOfWork,
            CancellationToken ct)
        {
            JsonArray? filesArr = null;
            try
            {
                var root = JsonNode.Parse(payloadJson)!.AsObject();
                filesArr = root["files"] as JsonArray;
            }
            catch
            {
                /* нет секции files */
            }

            var incoming = new Dictionary<Guid, (string fileName, string fileType)>();
            if (filesArr is not null)
            {
                foreach (var node in filesArr.OfType<JsonObject>())
                {
                    var fileIdStr = node["fileId"]?.GetValue<string>();
                    var fileName = node["fileName"]?.GetValue<string>();
                    var fileType = node["fileType"]?.GetValue<string>();

                    if (string.IsNullOrWhiteSpace(fileIdStr) || !Guid.TryParse(fileIdStr, out var fileId))
                        throw new HandlerException(
                            "Каждый объект в \"files\" должен иметь корректный GUID в поле fileId",
                            ErrorCodesEnum.Business);
                    if (string.IsNullOrWhiteSpace(fileName) || string.IsNullOrWhiteSpace(fileType))
                        throw new HandlerException(
                            "Каждый объект в \"files\" должен иметь непустые fileName и fileType",
                            ErrorCodesEnum.Business);

                    incoming[fileId] = (fileName!, fileType!);
                }
            }

            var existing = await unitOfWork.ProcessFileRepository.GetByFilterListAsync(
                ct, f => f.ProcessDataId == processDataId);

            var byFileId = existing
                .Where(e => e.FileId != Guid.Empty)
                .ToDictionary(e => e.FileId, e => e);

            var incomingIds = incoming.Keys.ToHashSet();

            // add/restore/rename
            foreach (var kv in incoming)
            {
                var fileId = kv.Key;
                var (fileName, fileType) = kv.Value;

                if (!byFileId.TryGetValue(fileId, out var ex))
                {
                    await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileCreatedEvent
                    {
                        EntityId = Guid.NewGuid(),
                        ProcessDataId = processDataId,
                        FileId = fileId,
                        FileName = fileName,
                        FileType = fileType
                    }, ct);
                }
                else if (ex.State != ProcessFileState.Active || ex.FileName != fileName || ex.FileType != fileType)
                {
                    await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileStatusChangedEvent
                    {
                        EntityId = ex.Id,
                        State = ProcessFileState.Active,
                        FileName = fileName,
                        FileType = fileType
                    }, ct);
                }
            }

            // soft delete
            foreach (var ex in byFileId.Values)
            {
                if (!incomingIds.Contains(ex.FileId) && ex.State != ProcessFileState.Removed)
                {
                    await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileStatusChangedEvent
                    {
                        EntityId = ex.Id,
                        State = ProcessFileState.Removed
                    }, ct);
                }
            }
        }


        private static string? ReadClassificationFromJson(string? json)
        {
            if (string.IsNullOrWhiteSpace(json)) return null;

            try
            {
                using var doc = JsonDocument.Parse(json);
                var root = doc.RootElement;

                static bool TryGetPropertyCaseInsensitive(JsonElement obj, string name, out JsonElement value)
                {
                    if (obj.ValueKind == JsonValueKind.Object)
                    {
                        foreach (var p in obj.EnumerateObject())
                        {
                            if (string.Equals(p.Name, name, StringComparison.OrdinalIgnoreCase))
                            {
                                value = p.Value;
                                return true;
                            }
                        }
                    }

                    value = default;
                    return false;
                }

                static string? Canon(string? s)
                {
                    if (string.IsNullOrWhiteSpace(s)) return null;
                    var n = s.Trim().ToLowerInvariant();
                    return n switch
                    {
                        "emergency" => "Emergency",
                        "standard" => "Standard",
                        "normal" => "Normal",
                        _ => null
                    };
                }

                // processData.analyst.classificationCode
                if (TryGetPropertyCaseInsensitive(root, "processData", out var pd) &&
                    TryGetPropertyCaseInsensitive(pd, "analyst", out var analyst) &&
                    TryGetPropertyCaseInsensitive(analyst, "classificationCode", out var cc) &&
                    cc.ValueKind == JsonValueKind.String)
                {
                    return Canon(cc.GetString());
                }

                // плоские варианты в корне
                string? flat =
                    (TryGetPropertyCaseInsensitive(root, "classification", out var c) &&
                     c.ValueKind == JsonValueKind.String
                        ? c.GetString()
                        : null)
                    ?? (TryGetPropertyCaseInsensitive(root, "itsmLevel", out var il) &&
                        il.ValueKind == JsonValueKind.String
                        ? il.GetString()
                        : null)
                    ?? (TryGetPropertyCaseInsensitive(root, "level", out var lvl) &&
                        lvl.ValueKind == JsonValueKind.String
                        ? lvl.GetString()
                        : null)
                    ?? (TryGetPropertyCaseInsensitive(root, "priority", out var pr) &&
                        pr.ValueKind == JsonValueKind.String
                        ? pr.GetString()
                        : null);

                if (flat is not null) return Canon(flat);

                // плоские варианты внутри processData (на всякий случай)
                if (pd.ValueKind == JsonValueKind.Object)
                {
                    flat =
                        (TryGetPropertyCaseInsensitive(pd, "classification", out var c2) &&
                         c2.ValueKind == JsonValueKind.String
                            ? c2.GetString()
                            : null)
                        ?? (TryGetPropertyCaseInsensitive(pd, "itsmLevel", out var il2) &&
                            il2.ValueKind == JsonValueKind.String
                            ? il2.GetString()
                            : null)
                        ?? (TryGetPropertyCaseInsensitive(pd, "level", out var lvl2) &&
                            lvl2.ValueKind == JsonValueKind.String
                            ? lvl2.GetString()
                            : null)
                        ?? (TryGetPropertyCaseInsensitive(pd, "priority", out var pr2) &&
                            pr2.ValueKind == JsonValueKind.String
                            ? pr2.GetString()
                            : null);

                    if (flat is not null) return Canon(flat);
                }
            }
            catch
            {
                /* ignore */
            }

            return null;
        }


        private static string? ReadClassificationLoose(Dictionary<string, object>? payload, string? storedPayloadJson)
        {
            string? fromIncoming = null;

            try
            {
                if (payload is not null)
                {
                    // формат: { payload: { processData: { analyst: { classificationCode } } } }
                    if (payload.TryGetValue("payload", out var pRaw) && pRaw is not null)
                    {
                        var p = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pRaw));
                        if (p is not null && p.TryGetValue("processData", out var pdRaw) && pdRaw is not null)
                        {
                            var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(
                                JsonSerializer.Serialize(pdRaw));
                            if (pd is not null && pd.TryGetValue("analyst", out var aRaw) && aRaw is not null)
                            {
                                var analyst =
                                    JsonSerializer.Deserialize<Dictionary<string, object>>(
                                        JsonSerializer.Serialize(aRaw));
                                if (analyst != null && analyst.TryGetValue("classificationCode", out var cRaw) &&
                                    cRaw is not null)
                                    fromIncoming = JsonSerializer.Deserialize<string>(JsonSerializer.Serialize(cRaw));
                            }
                        }
                    }

                    // формат: { processData: { analyst: { classificationCode } } }
                    if (fromIncoming is null && payload.TryGetValue("processData", out var pd2) && pd2 is not null)
                    {
                        var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pd2));
                        if (pd is not null && pd.TryGetValue("analyst", out var a2) && a2 is not null)
                        {
                            var analyst =
                                JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(a2));
                            if (analyst != null && analyst.TryGetValue("classificationCode", out var c2) &&
                                c2 is not null)
                                fromIncoming = JsonSerializer.Deserialize<string>(JsonSerializer.Serialize(c2));
                        }
                    }
                }
            }
            catch
            {
                /* ignore */
            }

            // fallback: берём из сохранённого payload процесса
            var fromStored = ReadClassificationFromJson(storedPayloadJson);

            static string? Canon(string? s)
            {
                if (string.IsNullOrWhiteSpace(s)) return null;
                var n = s.Trim().ToLowerInvariant();
                return n switch
                {
                    "emergency" => "Emergency",
                    "standard" => "Standard",
                    "normal" => "Normal",
                    _ => null
                };
            }

            return Canon(fromIncoming) ?? Canon(fromStored);
        }
    }
}
