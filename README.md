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
    /// ITSM Send (универсально):
    /// - Если в запросе пришёл processData и он отличается — обновить payload процесса (DB) и файлы
    /// - Для схемы с ProcessStage.Level1/2/3: шлем переменные в Camunda (Line1/Line2/…)
    /// - Остальная логика без изменений
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
                              ?? throw new HandlerException("Задача не найдена или уже обработана", ErrorCodesEnum.Business);

            var processData = await unitOfWork.ProcessDataRepository.GetByIdAsync(ct, currentTask.ProcessDataId)
                              ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

            var parentTask = await unitOfWork.ProcessTaskRepository.GetByIdAsync(ct, currentTask.ParentTaskId!.Value);

            // 2) Если прислали processData — апдейтим процесс (DB) и файлы, если реально меняется
            await UpdateProcessDataIfChangedAsync(processData, command.PayloadJson, unitOfWork, ct);

            if (string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
            {
                var claimed = await camundaService.CamundaClaimedTasks(
                                  processData.ProcessInstanceId, parentTask.BlockCode)
                              ?? throw new HandlerException(
                                     $"Не найдена задача в Camunda: {processData.ProcessInstanceId} {parentTask.BlockCode};",
                                     ErrorCodesEnum.Camunda);

                var variables = new Dictionary<string, object>();

                // ===== Вторая схема: Level1 / Level2 / Level3 =====
                if (Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var stage))
                {
                    if (stage == ProcessStage.Level1)
                    {
                        var code = ReadExecutionCode(command.PayloadJson, 1); // "executed"|"toSecondLine"|null
                        bool line1 = code?.Equals("executed", StringComparison.OrdinalIgnoreCase) == true
                                     ? true
                                     : code?.Equals("toSecondLine", StringComparison.OrdinalIgnoreCase) == true
                                         ? false
                                         : true; // дефолт
                        variables["Line1"] = line1; // если в BPMN с пробелом — переименуй на "Line 1"
                    }
                    else if (stage == ProcessStage.Level2)
                    {
                        var code = ReadExecutionCode(command.PayloadJson, 2); // объект {code,name} или строка
                        string? line2 = code switch
                        {
                            var c when c?.Equals("Executed",    StringComparison.OrdinalIgnoreCase) == true => "Executed",
                            var c when c?.Equals("ExternalOrg", StringComparison.OrdinalIgnoreCase) == true => "ExternalOrg",
                            var c when c?.Equals("Line3",       StringComparison.OrdinalIgnoreCase) == true
                                   || c?.Equals("toThirdLine", StringComparison.OrdinalIgnoreCase) == true => "Line3",
                            _ => null
                        };
                        if (!string.IsNullOrWhiteSpace(line2))
                            variables["Line2"] = line2!;
                    }
                    else if (stage == ProcessStage.Level3)
                    {
                        // при необходимости добавь Line3 аналогично
                        // var code = ReadExecutionCode(command.PayloadJson, 3);
                        // variables["Line3"] = ...;
                    }
                }
                // ===== /схема =====

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

                // Закрываем родителя (system)
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

        // ===== UPDATE processData секции, если изменилась =====

        private static async Task UpdateProcessDataIfChangedAsync(
            ProcessDataEntity processData,
            Dictionary<string, object>? incomingPayload,
            IUnitOfWork unitOfWork,
            CancellationToken ct)
        {
            var incomingProcessData = ExtractIncomingProcessData(incomingPayload);
            if (incomingProcessData is null) return; // нечего обновлять

            var merged = MergeProcessDataSection(processData.PayloadJson, incomingProcessData);

            if (!string.Equals(merged, processData.PayloadJson, StringComparison.Ordinal))
            {
                // используем штатное событие, чтобы не потерять обязательные поля (ProcessCode и т.п.)
                await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataEditedEvent
                {
                    EntityId      = processData.Id,
                    StatusCode    = processData.StatusCode,
                    StatusName    = processData.StatusName,
                    PayloadJson   = merged,
                    InitiatorCode = processData.InitiatorCode,
                    InitiatorName = processData.InitiatorName,
                    Title         = processData.Title
                }, ct);

                // синхронизация файлов (если файлы лежат в корне payload процесса)
                await ReconcileFilesAsync(processData.Id, merged, unitOfWork, ct);

                // локально держим актуальное значение
                processData.PayloadJson = merged;
            }
        }

        /// Достаём JsonObject processData из входного payload.
        private static JsonObject? ExtractIncomingProcessData(Dictionary<string, object>? payload)
        {
            if (payload is null) return null;

            // вариант 1: { payload: { processData: {...} } }
            if (payload.TryGetValue("payload", out var pRaw) && pRaw is not null)
            {
                var p = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pRaw));
                if (p != null && p.TryGetValue("processData", out var pdRaw) && pdRaw is not null)
                    return JsonNode.Parse(JsonSerializer.Serialize(pdRaw)) as JsonObject;
            }

            // вариант 2: { processData: {...} } прямо в payloadJson
            if (payload.TryGetValue("processData", out var pdRaw2) && pdRaw2 is not null)
                return JsonNode.Parse(JsonSerializer.Serialize(pdRaw2)) as JsonObject;

            return null;
        }

        /// Мерджим ТОЛЬКО секцию processData.
        private static string MergeProcessDataSection(string? baseJson, JsonObject incomingProcessData)
        {
            var root = (JsonNode.Parse(string.IsNullOrWhiteSpace(baseJson) ? "{}" : baseJson) as JsonObject) ?? new JsonObject();
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
                        target[kv.Key] = kv.Value.DeepClone(); // массивы/примитивы — overwrite
                    }
                }
            }
        }

        /// execution.code для levelN (строка или объект {code,name})
        private static string? ReadExecutionCode(Dictionary<string, object>? payload, int levelIndex)
        {
            try
            {
                if (payload is null) return null;

                if (!payload.TryGetValue("payload", out var pRaw) || pRaw is null)
                {
                    if (payload.TryGetValue("processData", out var flatPdRaw) && flatPdRaw is not null)
                        return FromProcessData(flatPdRaw, levelIndex);
                    return null;
                }

                var p = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pRaw));
                if (p is null || !p.TryGetValue("processData", out var pdRaw) || pdRaw is null) return null;

                return FromProcessData(pdRaw, levelIndex);

                static string? FromProcessData(object pdRaw, int idx)
                {
                    var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pdRaw));
                    var levelKey = $"level{idx}";
                    if (pd is null || !pd.TryGetValue(levelKey, out var lRaw) || lRaw is null) return null;

                    var level = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(lRaw));
                    if (level is null || !level.TryGetValue("formData", out var fdRaw) || fdRaw is null) return null;

                    var form = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(fdRaw));
                    if (form is null || !form.TryGetValue("execution", out var exRaw) || exRaw is null) return null;

                    if (exRaw is JsonElement je && je.ValueKind == JsonValueKind.String) // простая строка
                        return je.GetString();

                    var exDict = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(exRaw));
                    if (exDict != null && exDict.TryGetValue("code", out var codeRaw) && codeRaw != null)
                        return JsonSerializer.Deserialize<string>(JsonSerializer.Serialize(codeRaw));

                    return null;
                }
            }
            catch { return null; }
        }

        // ===== файлы (как в твоём рабочем коде) =====
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
            catch { /* no files */ }

            var incoming = new Dictionary<Guid, (string fileName, string fileType)>();
            if (filesArr is not null)
            {
                foreach (var node in filesArr.OfType<JsonObject>())
                {
                    var fileIdStr = node["fileId"]?.GetValue<string>();
                    var fileName  = node["fileName"]?.GetValue<string>();
                    var fileType  = node["fileType"]?.GetValue<string>();

                    if (string.IsNullOrWhiteSpace(fileIdStr) || !Guid.TryParse(fileIdStr, out var fileId))
                        throw new HandlerException("Каждый объект в \"files\" должен иметь корректный GUID в поле fileId",
                            ErrorCodesEnum.Business);
                    if (string.IsNullOrWhiteSpace(fileName) || string.IsNullOrWhiteSpace(fileType))
                        throw new HandlerException("Каждый объект в \"files\" должен иметь непустые fileName и fileType",
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

            // add / restore / rename
            foreach (var kv in incoming)
            {
                var fileId = kv.Key;
                var (fileName, fileType) = kv.Value;

                if (!byFileId.TryGetValue(fileId, out var ex))
                {
                    await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileCreatedEvent
                    {
                        EntityId      = Guid.NewGuid(),
                        ProcessDataId = processDataId,
                        FileId        = fileId,
                        FileName      = fileName,
                        FileType      = fileType
                    }, ct);
                }
                else if (ex.State != ProcessFileState.Active || ex.FileName != fileName || ex.FileType != fileType)
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

            // soft delete
            foreach (var ex in byFileId.Values)
            {
                if (!incomingIds.Contains(ex.FileId) && ex.State != ProcessFileState.Removed)
                {
                    await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileStatusChangedEvent
                    {
                        EntityId = ex.Id,
                        State    = ProcessFileState.Removed
                    }, ct);
                }
            }
        }
    }
}
