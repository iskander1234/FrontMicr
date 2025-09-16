using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Enum;
using BpmBaseApi.Shared.Models.Camunda;
using BpmBaseApi.Shared.Models.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using System.Text.Encodings.Web;
using System.Text.Json;
using System.Text.Json.Nodes;
using BpmBaseApi.Shared.Dtos;
using static BpmBaseApi.Shared.Models.Process.CommonData;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    public class StartProcessCommandHandler(
        IUnitOfWork unitOfWork,
        IPayloadReaderService payloadReader,
        IProcessTaskService helperService,
        ICamundaService camundaService)
        : IRequestHandler<StartProcessCommand, BaseResponseDto<StartProcessResponse>>
    {
        public async Task<BaseResponseDto<StartProcessResponse>> Handle(
            StartProcessCommand command,
            CancellationToken cancellationToken)
        {
            var process = await unitOfWork.ProcessRepository
                              .GetByFilterAsync(cancellationToken, p => p.ProcessCode == command.ProcessCode)
                          ?? throw new HandlerException($"Процесс с кодом {command.ProcessCode} не найден",
                              ErrorCodesEnum.Business);

            var jsonOptions = new JsonSerializerOptions { Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping };
            var payloadJson = JsonSerializer.Serialize(command.Payload, jsonOptions);

            var regData = payloadReader.ReadSection<RegData>(command.Payload, "regData");
            var processDataDto = payloadReader.ReadSection<ProcessDataDto>(command.Payload, "processData");

            // NEW: читаем номер из payload.regData.regnum
            var regnumFromPayload = TryGetRegnumFromPayload(payloadJson);

            // === A) если regnum задан, пробуем использовать существующую заявку ===
            if (!string.IsNullOrWhiteSpace(regnumFromPayload))
            {
                var alreadyStarted = await unitOfWork.ProcessDataRepository.GetByFilterAsync(
                    cancellationToken,
                    p => p.ProcessCode == process.ProcessCode
                         && p.RegNumber == regnumFromPayload
                         && p.StatusCode == "Started"
                );
                if (alreadyStarted is not null)
                {
                    return new BaseResponseDto<StartProcessResponse>
                    {
                        Data = new StartProcessResponse
                            { ProcessGuid = alreadyStarted.Id, RegNumber = alreadyStarted.RegNumber }
                    };
                }

                var draft = await unitOfWork.ProcessDataRepository.GetByFilterAsync(
                    cancellationToken,
                    p => p.ProcessCode == process.ProcessCode
                         && p.RegNumber == regnumFromPayload
                         && p.StatusCode == "Draft"
                );

                if (draft is not null)
                {
                    // NEW: синхронизируем payload.regData.regnum с реальным номером черновика
                    payloadJson = SetRegnumInPayload(payloadJson, draft.RegNumber, jsonOptions);
                    
                    // всегда фиксируем новый момент старта в payload
                    payloadJson = SetStartDateInPayload(payloadJson, DateTime.Now, jsonOptions);
                    await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataEditedEvent
                    {
                        EntityId = draft.Id,
                        StatusCode = "Started",
                        StatusName = "В работе",
                        PayloadJson = payloadJson,
                        InitiatorCode = regData?.UserCode,
                        InitiatorName = regData?.UserName,
                        Title = processDataDto?.DocumentTitle ?? draft.Title,
                        StartedAt = DateTime.Now
                        
                    }, cancellationToken);

                    await StartCamundaAndLinkAsync(draft.Id, cancellationToken);
                    await AddFilesUpsertAsync(draft.Id, payloadJson, cancellationToken);
                    await WriteHistoryStartIfAbsentAsync(draft.Id, draft.RegNumber, payloadJson, cancellationToken);

                    return new BaseResponseDto<StartProcessResponse>
                    {
                        Data = new StartProcessResponse { ProcessGuid = draft.Id, RegNumber = draft.RegNumber }
                    };
                }
            }

            // === B) черновика нет — создаём новую Started ===
            var requestNumber = await helperService.GenerateRequestNumberAsync(command.ProcessCode, cancellationToken);

            // NEW: вписываем номер в payload.regData.regnum
            payloadJson = SetRegnumInPayload(payloadJson, requestNumber, jsonOptions);
            payloadJson = SetStartDateInPayload(payloadJson, DateTime.Now, jsonOptions);
            var createdEvt = new ProcessDataCreatedEvent
            {
                ProcessId = process.Id,
                ProcessCode = process.ProcessCode,
                ProcessName = process.ProcessName,
                RegNumber = requestNumber,
                InitiatorCode = regData.UserCode,
                InitiatorName = regData.UserName,
                StatusCode = "Started",
                StatusName = "В работе",
                PayloadJson = payloadJson,
                Title = processDataDto.DocumentTitle,
                StartedAt = DateTime.Now
            };
            await unitOfWork.ProcessDataRepository.RaiseEvent(createdEvt, cancellationToken);

            await StartCamundaAndLinkAsync(createdEvt.EntityId, cancellationToken);
            await AddFilesUpsertAsync(createdEvt.EntityId, payloadJson, cancellationToken);
            await WriteHistoryStartIfAbsentAsync(createdEvt.EntityId, requestNumber, payloadJson, cancellationToken);

            return new BaseResponseDto<StartProcessResponse>
            {
                Data = new StartProcessResponse { ProcessGuid = createdEvt.EntityId, RegNumber = requestNumber }
            };

            // ---------- ЛОКАЛЬНЫЕ ФУНКЦИИ (NEW) ----------

            static string? TryGetRegnumFromPayload(string payload)
            {
                try
                {
                    var root = JsonNode.Parse(payload)!.AsObject();
                    return root["regData"]?["regnum"]?.GetValue<string>()?.Trim();
                }
                catch
                {
                    return null;
                }
            }

            static string SetRegnumInPayload(string payload, string regnum, JsonSerializerOptions opts)
            {
                var root = JsonNode.Parse(payload)!.AsObject();
                if (root["regData"] is not JsonObject rd)
                {
                    rd = new JsonObject();
                    root["regData"] = rd;
                }

                rd["regnum"] = regnum;
                return JsonSerializer.Serialize(root, opts);
            }

            async Task StartCamundaAndLinkAsync(Guid processDataId, CancellationToken ct)
            {
                var pi = await camundaService.CamundaStartProcess(new CamundaStartProcessRequest
                {
                    processCode = command.ProcessCode,
                    variables = new Dictionary<string, object> { { "processGuid", processDataId.ToString() } }
                });

                await unitOfWork.ProcessDataRepository.RaiseEvent(
                    new ProcessDataProcessInstanseIdChangedEvent
                    {
                        EntityId = processDataId,
                        ProcessInstanceId = pi
                    },
                    ct);
            }

            async Task AddFilesUpsertAsync(Guid processDataId, string payload, CancellationToken ct)
            {
                try
                {
                    var root = JsonNode.Parse(payload)!.AsObject();
                    var files = root["files"] as JsonArray;
                    if (files is null || files.Count == 0) return;

                    var existing = await unitOfWork.ProcessFileRepository.GetByFilterListAsync(
                        ct, f => f.ProcessDataId == processDataId);
                    var existingIds = existing.Where(x => x.FileId != Guid.Empty)
                        .Select(x => x.FileId)
                        .ToHashSet();

                    foreach (var node in files.OfType<JsonObject>())
                    {
                        var fileIdStr = node["fileId"]?.GetValue<string>();
                        var fileName = node["fileName"]?.GetValue<string>();
                        var fileType = node["fileType"]?.GetValue<string>();

                        if (string.IsNullOrWhiteSpace(fileName) || string.IsNullOrWhiteSpace(fileType))
                            throw new HandlerException(
                                "Каждый объект в \"files\" должен иметь непустые fileName и fileType",
                                ErrorCodesEnum.Business);
                        if (string.IsNullOrWhiteSpace(fileIdStr) || !Guid.TryParse(fileIdStr, out var fileId))
                            throw new HandlerException(
                                "Каждый объект в \"files\" должен иметь корректный GUID в поле fileId",
                                ErrorCodesEnum.Business);

                        if (existingIds.Contains(fileId)) continue;

                        await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileCreatedEvent
                        {
                            EntityId = Guid.NewGuid(),
                            ProcessDataId = processDataId,
                            FileId = fileId,
                            FileName = fileName,
                            FileType = fileType
                        }, ct);

                        existingIds.Add(fileId);
                    }
                }
                catch
                {
                    /* опционально залогировать */
                }
            }

            async Task WriteHistoryStartIfAbsentAsync(Guid processDataId, string regNumber, string payload,
                CancellationToken ct)
            {
                var hasStart = await unitOfWork.ProcessTaskHistoryRepository.CountAsync(
                    ct, h => h.ProcessDataId == processDataId && h.Action == ProcessAction.Start.ToString());
                if (hasStart > 0) return;

                await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(new ProcessTaskHistoryCreatedEvent
                {
                    ProcessDataId = processDataId,
                    TaskId = processDataId,
                    Action = ProcessAction.Start.ToString(),
                    BlockName = "Регистрационная форма",
                    Timestamp = DateTime.Now,
                    PayloadJson = payload,
                    Comment = "",
                    Description = "",
                    ProcessCode = command.ProcessCode,
                    ProcessName = process.ProcessName,
                    RegNumber = regNumber,
                    InitiatorCode = regData.UserCode,
                    InitiatorName = regData.UserName,
                    AssigneeCode = regData.UserCode,
                    AssigneeName = regData.UserName,
                    Title = processDataDto.DocumentTitle,
                    ActionName = "Зарегистрировано"
                }, ct);
            }
        }
        
        static string SetStartDateInPayload(string payload, DateTime dt, JsonSerializerOptions opts)
        {
            var root = JsonNode.Parse(payload)!.AsObject();
            if (root["regData"] is not JsonObject rd)
            {
                rd = new JsonObject();
                root["regData"] = rd;
            }

            // ISO 8601; если хочешь 'Z' — используй dt.ToUniversalTime().ToString("o")
            rd["startdate"] = dt.ToString("o");
            return JsonSerializer.Serialize(root, opts);
        }
    }
}

        

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
                                EntityId      = processData.Id,
                                StatusCode    = processData.StatusCode,
                                StatusName    = processData.StatusName,
                                PayloadJson   = mergedJson,
                                InitiatorCode = processData.InitiatorCode,
                                InitiatorName = processData.InitiatorName,
                                Title         = processData.Title
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

                await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode,
                    cancellationToken);
                await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

                // текущего — убрать из кэша его Pending (важно для positive пути)
                await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);
                
                // Был ли отрицательный исход на текущем шаге?
                bool isNegative = stage switch
                {
                    ProcessStage.Rework   => command.Action == ProcessAction.Cancel,
                    ProcessStage.Signing  => command.Action == ProcessAction.Reject,
                    _ /* Approval и прочее */ => command.Condition is ProcessCondition.remake or ProcessCondition.reject
                };
                
                if (isNegative)
                {

                    // текущего — убрать из кэша его Pending
                    await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

                    var affectedAssignees = siblings
                        .Where(t => t.Status == "Waiting" && !string.IsNullOrEmpty(t.AssigneeCode))
                        .Select(t => t.AssigneeCode)
                        .Distinct()
                        .ToList();

                    foreach (var w in siblings.Where(t => t.Status == "Waiting"))
                    {
                        await processTaskService.FinalizeTaskAsync(w, cancellationToken);
                    }

                    // у тех, у кого были waiting — почистить
                    foreach (var a in affectedAssignees)
                        await processTaskService.RefreshUserTaskCacheAsync(a, cancellationToken);
                }
                else
                {
                    // accept: продолжаем цепочку
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
                        {
                            Data = new SendProcessResponse { Success = true }
                        };
                    }

                    // сюда попадём, если очередников не осталось — пойдём сабмитить родителя как обычно
                }
            }


             var pendingSiblings = await unitOfWork.ProcessTaskRepository.CountAsync(
                     cancellationToken,
                     t => t.ParentTaskId == currentTask.ParentTaskId
                          && t.Id != currentTask.Id
                          && t.Status == "Pending");

            if (pendingSiblings > 0)
            {
                await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
                await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

                // текущего — убрать из кэша его Pending
                await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

                return new BaseResponseDto<SendProcessResponse>
                {
                    Data = new SendProcessResponse { Success = true }
                };
            }

            var parentTask =
                await unitOfWork.ProcessTaskRepository.GetByIdAsync(cancellationToken, currentTask.ParentTaskId.Value);

            if (parentTask.AssigneeCode == "system")
            {
                var claimedTasksResponse =
                    await camundaService.CamundaClaimedTasks(processData.ProcessInstanceId, parentTask.BlockCode)
                    ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);
                
                if (!Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var stageForCamunda))
                    throw new InvalidOperationException($"Unknown process stage: {currentTask.BlockCode}");

                // >>> НОВОЕ: получаем правильные булевы переменные под конкретный этап
                var variables = await BuildCamundaVariablesAsync(
                    stageForCamunda,
                    command.Condition, // accept/remake/reject
                    command.Action, // <<< ДОБАВИЛИ
                    processData.Id,
                    parentTask.Id, // нужен для Approval-раунда
                    unitOfWork,
                    cancellationToken);

                var submitResponse = await camundaService.CamundaSubmitTask(
                    new CamundaSubmitTaskRequest
                    {
                        TaskId = claimedTasksResponse.TaskId,
                        Variables = variables
                    });

                if (!submitResponse.Success)
                {
                    throw new HandlerException($"Ошибка при отправке Msg:{submitResponse.Msg}", ErrorCodesEnum.Camunda);
                }

                await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
                await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
                await processTaskService.FinalizeTaskAsync(parentTask, cancellationToken);

                // иначе у части пользователей висит старая карточка
                if (!string.IsNullOrWhiteSpace(currentTask.AssigneeCode))
                    await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode!, cancellationToken);

                return new BaseResponseDto<SendProcessResponse>
                {
                    Data = new SendProcessResponse { Success = true }
                };
            }
            
            //Добавил 
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

        private static string GetCamundaVarNameForStage(ProcessStage stage) => stage switch
        {
            ProcessStage.Approval => "agreement",
            ProcessStage.Signing => "sign",
            ProcessStage.Execution => "executed", // если в BPMN ждёшь другое имя — поменяй тут
            ProcessStage.ExecutionCheck => "checked", // при необходимости переименуй
            ProcessStage.Rework => "refinement", // <<< БЫЛО "agreement", теперь "refinement"
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

            var baseObj  = ParseOrEmpty(baseJson);
            var patchObj = ParseOrEmpty(patchJson);

            DeepMerge(baseObj, patchObj);
            return JsonSerializer.Serialize(baseObj, opts);

            static JsonObject ParseOrEmpty(string? json)
            {
                if (string.IsNullOrWhiteSpace(json)) return new JsonObject();
                try { return (JsonNode.Parse(json) as JsonObject) ?? new JsonObject(); }
                catch { return new JsonObject(); }
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
                        (rd.TryGetProperty("regnum", out var rn) || TryGetPropertyCaseInsensitive(rd, "regnum", out rn)))
                    {
                        return rn.ValueKind == JsonValueKind.String &&
                               !string.IsNullOrWhiteSpace(rn.GetString());
                    }
                }
            }
            catch { /* некорректный JSON — считаем, что regnum нет */ }

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
            // 1) Считываем итоговый список файлов из payload
            JsonArray? filesArr = null;
            try
            {
                var root = JsonNode.Parse(payloadJson)!.AsObject();
                filesArr = root["files"] as JsonArray;
            }
            catch
            {
                /* нет/битый files — молча выходим */
            }

            if (filesArr is null) return;

            var incoming = new Dictionary<Guid, (string fileName, string fileType)>();
            foreach (var node in filesArr.OfType<JsonObject>())
            {
                var fileIdStr = node["fileId"]?.GetValue<string>();
                var fileName = node["fileName"]?.GetValue<string>();
                var fileType = node["fileType"]?.GetValue<string>();

                if (string.IsNullOrWhiteSpace(fileIdStr) || !Guid.TryParse(fileIdStr, out var fileId))
                    throw new HandlerException("Каждый объект в \"files\" должен иметь корректный GUID в поле fileId",
                        ErrorCodesEnum.Business);
                if (string.IsNullOrWhiteSpace(fileName) || string.IsNullOrWhiteSpace(fileType))
                    throw new HandlerException("Каждый объект в \"files\" должен иметь непустые fileName и fileType",
                        ErrorCodesEnum.Business);

                incoming[fileId] = (fileName!, fileType!); // dedupe по fileId
            }

            // 2) Текущее состояние файлов в БД
            var existing = await unitOfWork.ProcessFileRepository.GetByFilterListAsync(
                ct, f => f.ProcessDataId == processDataId);

            var existingById = existing
                .Where(e => e.FileId != Guid.Empty)
                .ToDictionary(e => e.FileId, e => e);

            var incomingIds = incoming.Keys.ToHashSet();

            // 3) ADD: создаём отсутствующие в БД файлы
            foreach (var kv in incoming)
            {
                var fileId = kv.Key;
                var (fileName, fileType) = kv.Value;

                if (!existingById.ContainsKey(fileId))
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
                else
                {
                    // (опционально) если имя/тип поменялись — подними Update-событие, если оно есть
                    // var ex = existingById[fileId];
                    // if (ex.FileName != fileName || ex.FileType != fileType)
                    //     await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileUpdatedEvent { ... }, ct);
                }
            }

            // 4) DELETE: удаляем те, которых больше нет в итоговом payload
            foreach (var ex in existingById.Values)
            {
                if (!incomingIds.Contains(ex.FileId))
                {
                    await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileDeletedEvent
                    {
                        EntityId = ex.Id,
                        // ProcessDataId = processDataId // если событие его поддерживает — добавь
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

using BpmBaseApi.Domain.Entities.Event.Document;
using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.SeedWork;

namespace BpmBaseApi.Domain.Entities.Process;

/// <summary>
/// Файлы, прикреплённые к заявке (ProcessData)
/// </summary>
public class ProcessFileEntity : BaseJournaledEntity
{
    /// <summary>PK – GUID файла</summary>
    //public Guid Id { get; set; }
    public Guid FileId { get; set; }
    /// <summary>Имя файла</summary>
    public string FileName { get; set; }

    /// <summary>Тип файла (расширение или MIME)</summary>
    public string FileType { get; set; }

    /// <summary>FK → ProcessData.Id</summary>
    public Guid ProcessDataId { get; set; }

    // Навигация (необязательно, но полезно)
    public ProcessDataEntity ProcessData { get; set; }
    
    // EF + Журнал требуют публичный конструктор без параметров
    public ProcessFileEntity() { }

    #region Apply

    /// <summary>Заполнение полей при ProcessFileCreatedEvent</summary>
    public void Apply(ProcessFileCreatedEvent @event)
    {
        // @event.EntityId берётся из RaiseEvent(...).EntityId
        Id            = @event.EntityId;
        FileId        = @event.FileId;         // <-- NEW
        FileName      = @event.FileName;
        FileType      = @event.FileType;
        ProcessDataId = @event.ProcessDataId;
    }

    public void Apply(SignDocumentCreateEvent @event)
    {
        // @event.EntityId берётся из RaiseEvent(...).EntityId
        Id = @event.EntityId;
        FileId = @event.FileId;         // <-- NEW
        FileName = @event.FileName;
        FileType = @event.FileType;
        ProcessDataId = @event.ProcessDataId;
    }

    /// <summary>При удалении файла ничего не нужно делать —
    /// репозиторий вызовет DeleteByIdAsync автоматически</summary>
    // public void Apply(ProcessFileDeletedEvent @event)
    // {
    //     // пусто
    // }

    #endregion
}
