Я добавил новое   ProcessStage.Registration => "registrar",

private static string GetCamundaVarNameForStage(ProcessStage stage) => stage switch
        {
            ProcessStage.Approval => "agreement",
            ProcessStage.Signing => "sign",
            ProcessStage.Execution => "executed", // если в BPMN ждёшь другое имя — поменяй тут
            ProcessStage.ExecutionCheck => "checked", // при необходимости переименуй
            ProcessStage.Rework => "refinement", // <<< БЫЛО "agreement", теперь "refinement"
            ProcessStage.Registration => "registrar",
            _ => "agreement" // безопасный дефолт
        };

        надос сделать так если registrar Submit то дальше true а если  Reject то надо будет false и на доработку 

        и ттаже еще 1 ProcessStage.ExecutionCheck => "executionCheck", // при необходимости переименуй я переминовал теперь это executionCheck true и false ессть 
      

        using System.Text.Encodings.Web;
using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Enum;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using System.Text.Json;
using System.Text.Json.Nodes;
using AutoMapper;
using Microsoft.Extensions.Caching.Memory;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Models.Process;
using BpmBaseApi.Shared.Models.Camunda;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    public class SendProcessCommandHandler(
        IMapper mapper,
        IUnitOfWork unitOfWork,
        IMemoryCache cache,
        ICamundaService camundaService,
        IProcessTaskService processTaskService)
        : IRequestHandler<SendProcessCommand, BaseResponseDto<SendProcessResponse>>
    {
        public async Task<BaseResponseDto<SendProcessResponse>> Handle(SendProcessCommand command,
            CancellationToken cancellationToken)
        {
            // Берём задачу без фильтра по статусу — чтобы поймать Waiting и вернуть 400 с подсказкой
            var currentTask = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                cancellationToken,
                p => p.Id == command.TaskId
            ) ?? throw new HandlerException("Задача не найдена", ErrorCodesEnum.Business);

            var processData =
                await unitOfWork.ProcessDataRepository.GetByIdAsync(cancellationToken, currentTask.ProcessDataId)
                ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

            // Rework → Submit: апдейтим ProcessData ТОЛЬКО если в payload есть regData.regnum
            if (Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var currentStage)
                && currentStage == ProcessStage.Rework
                && command.Action == ProcessAction.Submit)
            {
                // payload может не прийти — тогда апдейт просто пропускаем
                if (command.PayloadJson is not null)
                {
                    var incomingJson = JsonSerializer.Serialize(command.PayloadJson);

                    // ← ключевое условие: обновляем только если есть regData.regnum
                    if (JsonHasRegnum(incomingJson))
                    {
                        var mergedJson = MergePayloadPreserve(processData.PayloadJson, incomingJson);

                        // апдейтим только при реальных изменениях
                        if (!string.Equals(mergedJson, processData.PayloadJson, StringComparison.Ordinal))
                        {
                            await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataEditedEvent
                            {
                                EntityId = processData.Id,
                                StatusCode = processData.StatusCode,
                                StatusName = processData.StatusName,
                                PayloadJson = mergedJson,
                                InitiatorCode = processData.InitiatorCode,
                                InitiatorName = processData.InitiatorName,
                                Title = processData.Title
                                // StartedAt не трогаем
                            }, cancellationToken);

                            // синхронизируем таблицу файлов под новый payload
                            await ReconcileFilesAsync(processData.Id, mergedJson, unitOfWork, cancellationToken);

                            // держим актуальную версию локально, чтобы ниже читалась новая
                            processData.PayloadJson = mergedJson;
                        }
                    }
                    // если regnum нет — апдейт пропускаем, идём дальше по вашей логике
                }
            }


            // Последовательный ли режим из processData.approvalTypeCode
            var pdPayload = JsonSerializer.Deserialize<Dictionary<string, object>>(processData.PayloadJson);
            bool isSequential = ReadIsSequentialFromProcessData(pdPayload);

            // Если Sequentially и задача НЕ Pending → 400 и сказать кто Pending
            if (isSequential && !string.Equals(currentTask.Status, "Pending", StringComparison.OrdinalIgnoreCase))
            {
                ProcessTaskEntity? activePending = null;

                if (currentTask.ParentTaskId != null)
                {
                    activePending = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                        cancellationToken,
                        t => t.ParentTaskId == currentTask.ParentTaskId && t.Status == "Pending"
                    );
                }

                if (activePending == null)
                {
                    activePending = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                        cancellationToken,
                        t => t.ProcessDataId == currentTask.ProcessDataId
                             && t.BlockCode == currentTask.BlockCode
                             && t.Status == "Pending"
                    );
                }

                var responsible = activePending?.AssigneeCode ?? "не найден";
                throw new HandlerException(
                    $"Отправка запрещена: сначала должен отправить исполнитель со статусом Pending ({responsible}).",
                    ErrorCodesEnum.Business // у тебя это маппится в HTTP 400
                );
            }

            // Старая проверка «только Pending» (для параллельного режима как было)
            if (!string.Equals(currentTask.Status, "Pending", StringComparison.OrdinalIgnoreCase))
                throw new HandlerException("Задача не найдена или уже обработана", ErrorCodesEnum.Business);

            var recipients = ExtractFromPayload<List<UserInfo>>(command.PayloadJson, "recipients") ?? new();
            //var comment = ExtractFromPayload<string>(command.PayloadJson, "comment");

            // Делегирование
            if (command.Action == ProcessAction.Delegate && recipients.Any())
            {
                await processTaskService.HandleDelegationAsync(currentTask, processData, command, recipients,
                    cancellationToken);
                return new BaseResponseDto<SendProcessResponse>
                {
                    Data = new SendProcessResponse { Success = true }
                };
            }

            // Если sequential — закрываем текущего. Если negative → чистим остальных и идём к Camunda.
            // Если positive (accept) → поднимаем следующего Waiting -> Pending и выходим.
            if (isSequential)
            {
                if (!Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var stage))
                    throw new InvalidOperationException($"Unknown process stage: {currentTask.BlockCode}");

                var parent = await unitOfWork.ProcessTaskRepository
                    .GetByIdAsync(cancellationToken, currentTask.ParentTaskId!.Value);

                var siblings = await unitOfWork.ProcessTaskRepository
                    .GetByFilterListAsync(cancellationToken, t => t.ParentTaskId == parent.Id);

                // 1) лог + закрываем текущего (ОДИН РАЗ)
                await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode,
                    cancellationToken);
                await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
                // текущего — убрать из кэша его Pending (важно для positive пути)
                await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

                // 2) определяем исход шага
                bool isNegative = stage switch
                {
                    ProcessStage.Rework => command.Action == ProcessAction.Cancel,
                    ProcessStage.Signing => command.Action == ProcessAction.Reject,
                    _ => command.Condition is ProcessCondition.remake or ProcessCondition.reject
                };

                if (isNegative)
                {
                    // закрываем все Waiting в этом раунде
                    var affectedAssignees = siblings
                        .Where(t => t.Status == "Waiting" && !string.IsNullOrEmpty(t.AssigneeCode))
                        .Select(t => t.AssigneeCode)
                        .Distinct()
                        .ToList();

                    foreach (var w in siblings.Where(t => t.Status == "Waiting"))
                        await processTaskService.FinalizeTaskAsync(w, cancellationToken);

                    // у тех, у кого были waiting — почистить
                    foreach (var a in affectedAssignees)
                        await processTaskService.RefreshUserTaskCacheAsync(a, cancellationToken);

                    // завершаем раунд (Camunda/родитель) и ВОЗВРАЩАЕМСЯ
                    await HandleEndOfSequentialRoundAsync(parent, processData, stage, command, cancellationToken);
                    return new BaseResponseDto<SendProcessResponse>
                        { Data = new SendProcessResponse { Success = true } };
                }
                else
                {
                    // positive: ищем следующего
                    var next = siblings
                        .Where(t => t.Status == "Waiting")
                        .OrderBy(t => t.Order ?? int.MaxValue)
                        .ThenBy(t => t.Created)
                        .FirstOrDefault();

                    if (next != null)
                    {
                        await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
                        {
                            EntityId = next.Id,
                            Status = "Pending"
                        }, cancellationToken);

                        // у next теперь новая Pending — обновим его кэш
                        if (!string.IsNullOrEmpty(next.AssigneeCode))
                            await processTaskService.RefreshUserTaskCacheAsync(next.AssigneeCode, cancellationToken);

                        return new BaseResponseDto<SendProcessResponse>
                            { Data = new SendProcessResponse { Success = true } };
                    }

                    // «следующего» нет — раунд завершён: Camunda/родитель и ВОЗВРАЩАЕМСЯ
                    await HandleEndOfSequentialRoundAsync(parent, processData, stage, command, cancellationToken);
                    return new BaseResponseDto<SendProcessResponse>
                        { Data = new SendProcessResponse { Success = true } };
                }
            }

            var pendingSiblings = await unitOfWork.ProcessTaskRepository.CountAsync(
                cancellationToken,
                t => t.ParentTaskId == currentTask.ParentTaskId
                     && t.Id != currentTask.Id
                     && t.Status == "Pending");

            if (pendingSiblings > 0)
            {
                await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode,
                    cancellationToken);
                await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
                
                // текущего — убрать из кэша его Pending
                await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

                return new BaseResponseDto<SendProcessResponse>
                {
                    Data = new SendProcessResponse { Success = true }
                };
            }

            var parentTask = await unitOfWork.ProcessTaskRepository
                .GetByIdAsync(cancellationToken, currentTask.ParentTaskId!.Value);

            if (parentTask.AssigneeCode == "system")
            {
                var claimedTasksResponse =
                    await camundaService.CamundaClaimedTasks(processData.ProcessInstanceId, parentTask.BlockCode)
                    ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);

                if (!Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var stageForCamunda))
                    throw new InvalidOperationException($"Unknown process stage: {currentTask.BlockCode}");

                // получаем правильные булевы переменные под конкретный этап
                var variables = await BuildCamundaVariablesAsync(
                    stageForCamunda, command.Condition, command.Action,
                    processData.Id, parentTask.Id, unitOfWork, cancellationToken);

                var submitResponse = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
                {
                    TaskId = claimedTasksResponse.TaskId,
                    Variables = variables
                });

                if (!submitResponse.Success)
                    throw new HandlerException($"Ошибка при отправке Msg:{submitResponse.Msg}", ErrorCodesEnum.Camunda);

                await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode,
                    cancellationToken);
                await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
                await processTaskService.FinalizeTaskAsync(parentTask, cancellationToken);
                
                // иначе у части пользователей висит старая карточка
                if (!string.IsNullOrWhiteSpace(currentTask.AssigneeCode))
                    await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode!, cancellationToken);

                return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };
            }

            await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
            await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
            // финализировали current выше — теперь обновим его кэш один раз
            await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);
            // поднимем родителя
            await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
            {
                EntityId = parentTask.Id,
                Status = "Pending"
            }, cancellationToken);

            // у родителя появился Pending — обновим его кэш (если реальный исполнитель)
            if (!string.IsNullOrWhiteSpace(parentTask.AssigneeCode) &&
                !string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
            {
                await processTaskService.RefreshUserTaskCacheAsync(parentTask.AssigneeCode, cancellationToken);
            }

            return new BaseResponseDto<SendProcessResponse>
            {
                Data = new SendProcessResponse { Success = true }
            };
        }
        async Task HandleEndOfSequentialRoundAsync(
            ProcessTaskEntity parentTask,
            ProcessDataEntity processData,
            ProcessStage stageForCamunda,
            SendProcessCommand command,
            CancellationToken ct)
        {
            // если родитель — system, отправляем в Camunda
            if (string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
            {
                var claimedTasksResponse =
                    await camundaService.CamundaClaimedTasks(processData.ProcessInstanceId, parentTask.BlockCode)
                    ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);

                var variables = await BuildCamundaVariablesAsync(
                    stageForCamunda, command.Condition, command.Action,
                    processData.Id, parentTask.Id, unitOfWork, ct);

                var submitResponse = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
                {
                    TaskId = claimedTasksResponse.TaskId,
                    Variables = variables
                });

                if (!submitResponse.Success)
                    throw new HandlerException($"Ошибка при отправке Msg:{submitResponse.Msg}", ErrorCodesEnum.Camunda);

                await processTaskService.FinalizeTaskAsync(parentTask, ct);
            }
            else
            {
                // иначе — поднимаем родителя в Pending
                await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
                {
                    EntityId = parentTask.Id,
                    Status = "Pending"
                }, ct);

                if (!string.IsNullOrWhiteSpace(parentTask.AssigneeCode))
                    await processTaskService.RefreshUserTaskCacheAsync(parentTask.AssigneeCode, ct);
            }
        }
        private static string GetCamundaVarNameForStage(ProcessStage stage) => stage switch
        {
            ProcessStage.Approval => "agreement",
            ProcessStage.Signing => "sign",
            ProcessStage.Execution => "executed", // если в BPMN ждёшь другое имя — поменяй тут
            ProcessStage.ExecutionCheck => "checked", // при необходимости переименуй
            ProcessStage.Rework => "refinement", // <<< БЫЛО "agreement", теперь "refinement"
            ProcessStage.Registration => "registrar",
            _ => "agreement" // безопасный дефолт
        };

      

        private static async Task<Dictionary<string, object>> BuildCamundaVariablesAsync(
            ProcessStage stage,
            ProcessCondition condition,
            ProcessAction action,
            Guid processDataId,
            Guid? parentTaskId,
            IUnitOfWork unitOfWork,
            CancellationToken ct)
        {
            var varName = GetCamundaVarNameForStage(stage);

            switch (stage)
            {
                case ProcessStage.Rework:
                    // refinement: Submit => true, Cancel => false
                    return new Dictionary<string, object>
                    {
                        [varName] = (action == ProcessAction.Submit)
                    };

                case ProcessStage.Signing:
                    // sign: Submit => true, Reject => false
                    return new Dictionary<string, object>
                    {
                        [varName] = (action == ProcessAction.Submit)
                    };

                case ProcessStage.Approval:
                {
                    // agreement: accept => true, remake/reject => false
                    var isPositive = (condition == ProcessCondition.accept);

                    if (isPositive)
                    {
                        // «вето» в рамках того же родителя (раунда)
                        isPositive = await IsStagePositiveConsideringHistoryAsync(
                            ProcessStage.Approval, isPositive, processDataId, parentTaskId, unitOfWork, ct);
                    }

                    return new Dictionary<string, object> { [varName] = isPositive };
                }
                //  НОВОЕ: Execution — считаем позитивом и по Action, если Condition не задан
                case ProcessStage.Execution:
                {
                    bool isPositive = condition switch
                    {
                        ProcessCondition.accept => true,
                        ProcessCondition.remake or ProcessCondition.reject => false,
                        _ => action == ProcessAction.Submit
                    };
                    return new Dictionary<string, object> { [varName] = isPositive }; // varName == "executed"
                }

                //  НОВОЕ: ExecutionCheck — аналогично
                case ProcessStage.ExecutionCheck:
                {
                    bool isPositive = condition switch
                    {
                        ProcessCondition.accept => true,
                        ProcessCondition.remake or ProcessCondition.reject => false,
                        _ => action == ProcessAction.Submit
                    };
                    return new Dictionary<string, object> { [varName] = isPositive }; // varName == "checked"
                }


                default:
                {
                    // дефолт: accept => true
                    var isPositive = (condition == ProcessCondition.accept);
                    return new Dictionary<string, object> { [varName] = isPositive };
                }
            }
        }

        /// <summary>
        /// true, если processData.approvalTypeCode == "Sequentially"
        /// </summary>
        private static bool ReadIsSequentialFromProcessData(Dictionary<string, object>? payload)
        {
            if (payload == null) return false;
            try
            {
                if (!payload.TryGetValue("processData", out var pdRaw) || pdRaw is null) return false;

                var json = JsonSerializer.Serialize(pdRaw);
                var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(json);

                if (pd != null && pd.TryGetValue("approvalTypeCode", out var typeRaw) && typeRaw is not null)
                {
                    var tJson = JsonSerializer.Serialize(typeRaw);
                    var type = JsonSerializer.Deserialize<string>(tJson);
                    return string.Equals(type, "Sequentially", StringComparison.OrdinalIgnoreCase);
                }
            }
            catch
            {
                /* считаем Parallel */
            }

            return false;
        }

        private static async Task<bool> IsStagePositiveConsideringHistoryAsync(
            ProcessStage stage,
            bool currentIsPositive,
            Guid processDataId,
            Guid? parentTaskId,
            IUnitOfWork unitOfWork,
            CancellationToken ct)
        {
            if (!currentIsPositive) return false;

            var stageCode = stage.ToString();

            var negativeCount = await unitOfWork.ProcessTaskHistoryRepository.CountAsync(
                ct,
                h => h.ProcessDataId == processDataId
                     && h.BlockCode == stageCode
                     && h.ParentTaskId == parentTaskId
                     && (h.Condition == nameof(ProcessCondition.remake)
                         || h.Condition == nameof(ProcessCondition.reject)));

            return negativeCount == 0;
        }

        private static string MergePayloadPreserve(string baseJson, string patchJson)
        {
            var opts = new JsonSerializerOptions { Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping };

            var baseObj = ParseOrEmpty(baseJson);
            var patchObj = ParseOrEmpty(patchJson);

            DeepMerge(baseObj, patchObj);
            return JsonSerializer.Serialize(baseObj, opts);

            static JsonObject ParseOrEmpty(string? json)
            {
                if (string.IsNullOrWhiteSpace(json)) return new JsonObject();
                try
                {
                    return (JsonNode.Parse(json) as JsonObject) ?? new JsonObject();
                }
                catch
                {
                    return new JsonObject();
                }
            }

            static void DeepMerge(JsonObject target, JsonObject patch)
            {
                foreach (var kv in patch)
                {
                    if (kv.Value is null)
                    {
                        // null в patch — перезаписываем как null (или можно пропускать, если так задумано)
                        target[kv.Key] = null;
                        continue;
                    }

                    if (kv.Value is JsonObject patchObj)
                    {
                        if (target[kv.Key] is JsonObject targetObj)
                        {
                            DeepMerge(targetObj, patchObj);
                        }
                        else
                        {
                            target[kv.Key] = patchObj.DeepClone();
                        }
                    }
                    else
                    {
                        // массивы и примитивы — заменяем целиком
                        target[kv.Key] = kv.Value.DeepClone();
                    }
                }
            }
        }
        
        private static bool JsonHasRegnum(string json)
        {
            try
            {
                using var doc = JsonDocument.Parse(json);
                var root = doc.RootElement;

                if (root.ValueKind != JsonValueKind.Object) return false;

                if (root.TryGetProperty("regData", out var rd) ||
                    TryGetPropertyCaseInsensitive(root, "regData", out rd))
                {
                    if (rd.ValueKind == JsonValueKind.Object &&
                        (rd.TryGetProperty("regnum", out var rn) ||
                         TryGetPropertyCaseInsensitive(rd, "regnum", out rn)))
                    {
                        return rn.ValueKind == JsonValueKind.String &&
                               !string.IsNullOrWhiteSpace(rn.GetString());
                    }
                }
            }
            catch
            {
                /* некорректный JSON — считаем, что regnum нет */
            }

            return false;
        }

        private static bool TryGetPropertyCaseInsensitive(JsonElement element, string name, out JsonElement value)
        {
            if (element.ValueKind == JsonValueKind.Object)
            {
                foreach (var p in element.EnumerateObject())
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


        private static async Task ReconcileFilesAsync(
            Guid processDataId,
            string payloadJson,
            IUnitOfWork unitOfWork,
            CancellationToken ct)
        {
            // 1) Собираем итоговый список из payload
            JsonArray? filesArr = null;
            try
            {
                var root = JsonNode.Parse(payloadJson)!.AsObject();
                filesArr = root["files"] as JsonArray;
            }
            catch
            {
                /* нет секции files — выходим */
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

            // 2) Берём все файлы из БД (и 0 и 1)
            var existing = await unitOfWork.ProcessFileRepository.GetByFilterListAsync(
                ct, f => f.ProcessDataId == processDataId);

            var byFileId = existing
                .Where(e => e.FileId != Guid.Empty)
                .ToDictionary(e => e.FileId, e => e);

            var incomingIds = incoming.Keys.ToHashSet();

            // 3) ADD/UNDELETE/RENAME
            foreach (var kv in incoming)
            {
                var fileId = kv.Key;
                var (fileName, fileType) = kv.Value;

                if (!byFileId.TryGetValue(fileId, out var ex))
                {
                    // совсем новый
                    await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileCreatedEvent
                    {
                        EntityId = Guid.NewGuid(),
                        ProcessDataId = processDataId,
                        FileId = fileId,
                        FileName = fileName,
                        FileType = fileType
                    }, ct);
                }
                else
                {
                    // был, но мог быть удалён или изменились имя/тип
                    if (ex.State != ProcessFileState.Active || ex.FileName != fileName || ex.FileType != fileType)
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
            }

            // 4) SOFT DELETE: всё, чего нет в payload → State = Removed (1)
            foreach (var ex in byFileId.Values)
            {
                if (!incomingIds.Contains(ex.FileId) && ex.State != ProcessFileState.Removed)
                {
                    // Можно использовать старый ProcessFileDeletedEvent, он теперь ставит State=Removed
                    // await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileDeletedEvent { EntityId = ex.Id }, ct);

                    // Предпочтительно: единое событие статуса
                    await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileStatusChangedEvent
                    {
                        EntityId = ex.Id,
                        State = ProcessFileState.Removed
                    }, ct);
                }
            }
        }

        public static T? ExtractFromPayload<T>(Dictionary<string, object>? payload, string key)
        {
            if (payload == null || !payload.TryGetValue(key, out var value))
                return default;

            var json = JsonSerializer.Serialize(value);
            return JsonSerializer.Deserialize<T>(json, new JsonSerializerOptions
            {
                PropertyNameCaseInsensitive = true
            });
        }
    }
}
