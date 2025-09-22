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
    /// - ВАЖНО: если нашли классификацию — кладём process variable "classification" (Emergency|Standard|Normal)
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

            // 3) Переменные Camunda (plain key/value)
            var vars = new Dictionary<string, object?>(StringComparer.Ordinal)
            {
                ["processGuid"]   = created.EntityId.ToString(),
                ["initiatorCode"] = string.IsNullOrWhiteSpace(command.InitiatorCode) ? null : command.InitiatorCode,
                ["initiatorName"] = string.IsNullOrWhiteSpace(command.InitiatorName) ? null : command.InitiatorName
            };

            // КЛЮЧЕВОЕ: попробуем найти classification и положим в переменную
            var classification = ReadClassification(payloadJson);
            if (!string.IsNullOrWhiteSpace(classification))
                vars["classification"] = classification;

            var cleaned = vars.Where(kv => kv.Value is not null)
                              .ToDictionary(kv => kv.Key, kv => kv.Value!);

            if (cleaned.Values.Any(v => v is IDictionary<string, object>))
                throw new HandlerException("Camunda variables must be plain key/value.", ErrorCodesEnum.Business);

            // 4) Старт процесса
            var pi = await camundaService.CamundaStartProcess(new CamundaStartProcessRequest
            {
                processCode = command.ProcessCode,
                variables   = cleaned
                // businessKey = created.EntityId.ToString(), // включить, если поддерживается
            });

            await unitOfWork.ProcessDataRepository.RaiseEvent(
                new ProcessDataProcessInstanseIdChangedEvent
                {
                    EntityId          = created.EntityId,
                    ProcessInstanceId = pi
                },
                ct);

            // 5) Апсертим files из payload (если есть)
            await UpsertFilesFromPayloadAsync(created.EntityId, payloadJson, unitOfWork, ct);

            // 6) История "Start"
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
                ProcessCode   = process.ProcessCode,
                ProcessName   = process.ProcessName,
                RegNumber     = regnum,
                InitiatorCode = command.InitiatorCode,
                InitiatorName = command.InitiatorName,
                AssigneeCode  = command.InitiatorCode,
                AssigneeName  = command.InitiаторName,
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

        // Читаем classification из payload: processData.analyst.classificationCode | classification | level | priority
        static string? ReadClassification(string json)
        {
            var lvl = TryRead<string>(json, "processData", "analyst", "classificationCode")
                      ?? TryRead<string>(json, "processData", "classification")
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
                // логирование при желании
            }
        }
    }
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
using BpmBaseApi.Shared.Enum;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    /// <summary>
    /// ITSM Send:
    /// - читает classification из payloadJson (processData.analyst.classificationCode | classification | level | priority)
    /// - при системном parent: шлёт ВМЕСТЕ со stage-булевыми переменными (как старый Send) => gateway видит ${classification == 'Normal'}
    /// - иначе поднимаем родителя в Pending
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
            // Текущая подзадача должна быть Pending
            var currentTask = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                                  ct, t => t.Id == command.TaskId && t.Status == "Pending")
                              ?? throw new HandlerException("Задача не найдена или уже обработана",
                                  ErrorCodesEnum.Business);

            var processData = await unitOfWork.ProcessDataRepository.GetByIdAsync(ct, currentTask.ProcessDataId)
                              ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

            var parentTask = await unitOfWork.ProcessTaskRepository.GetByIdAsync(ct, currentTask.ParentTaskId!.Value);

            if (string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
            {
                var claimed = await camundaService.CamundaClaimedTasks(
                                  processData.ProcessInstanceId, parentTask.BlockCode)
                              ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);

                // 1) булевые для этапа (как в старом Send)
                var stage = Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var st) ? st : ProcessStage.Approval;
                var stageVars = await BuildCamundaVariablesAsync(
                    stage, command.Condition, command.Action,
                    processData.Id, parentTask.Id, unitOfWork, ct);

                // 2) строковая classification
                var classVars = BuildClassificationVariables(command.PayloadJson);

                // 3) объединяем
                foreach (var kv in classVars) stageVars[kv.Key] = kv.Value;

                var submit = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
                {
                    TaskId = claimed.TaskId,
                    Variables = stageVars
                });

                if (!submit.Success)
                    throw new HandlerException($"Ошибка при отправке Msg:{submit.Msg}", ErrorCodesEnum.Camunda);

                await processTaskService.FinalizeTaskAsync(parentTask, ct);
            }
            else
            {
                // Родитель не system — делаем его Pending
                await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
                {
                    EntityId = parentTask.Id,
                    Status = "Pending"
                }, ct);

                if (!string.IsNullOrWhiteSpace(parentTask.AssigneeCode))
                    await processTaskService.RefreshUserTaskCacheAsync(parentTask.AssigneeCode, ct);
            }

            // Лог + закрытие текущей подзадачи
            await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, ct);
            await processTaskService.FinalizeTaskAsync(currentTask, ct);
            await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, ct);

            return new BaseResponseDto<SendProcessResponse>
            {
                Data = new SendProcessResponse { Success = true }
            };
        }

        /// <summary>
        /// Ищем classification в payload (вложенность поддерживается).
        /// Возвращаем {"classification": "Emergency|Standard|Normal"} либо пустой словарь.
        /// </summary>
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
                "standard"  => "Standard",
                "normal"    => "Normal",
                _ => null
            };

            return canonical is null ? new() : new() { ["classification"] = canonical };
        }

        // ==== Вспомогалки для этапных булевых, как в старом Send ====

        private static string GetCamundaVarNameForStage(ProcessStage stage) => stage switch
        {
            ProcessStage.Approval       => "agreement",
            ProcessStage.Signing        => "sign",
            ProcessStage.Execution      => "executed",
            ProcessStage.ExecutionCheck => "executionCheck",
            ProcessStage.Rework         => "refinement",
            ProcessStage.Registration   => "registrar",
            _                           => "agreement"
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
            static bool? FromCondition(ProcessCondition c) => c switch
            {
                ProcessCondition.accept => true,
                ProcessCondition.remake or ProcessCondition.reject => false,
                _ => (bool?)null
            };

            static bool? FromAction(ProcessAction a) => a switch
            {
                ProcessAction.Submit => true,
                ProcessAction.Reject => false,
                ProcessAction.Cancel => false,
                _ => (bool?)null
            };

            // В Execution в BPMN нет условия — переменные не шлём вовсе.
            if (stage == ProcessStage.Execution)
                return new Dictionary<string, object>();

            var varName = GetCamundaVarNameForStage(stage);

            bool conditionDriven = stage is
                ProcessStage.Approval
                or ProcessStage.Signing
                or ProcessStage.ExecutionCheck
                or ProcessStage.Rework
                or ProcessStage.Registration;

            bool isPositive;

            if (conditionDriven)
            {
                var byCond = FromCondition(condition);
                var byAct  = FromAction(action);
                isPositive = byCond ?? byAct ?? false;

                if (stage == ProcessStage.Approval && isPositive)
                {
                    isPositive = await IsStagePositiveConsideringHistoryAsync(
                        ProcessStage.Approval, isPositive, processDataId, parentTaskId, unitOfWork, ct);
                }
            }
            else
            {
                isPositive = FromAction(action) ?? false;
            }

            return new Dictionary<string, object> { [varName] = isPositive };
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
    }
}
