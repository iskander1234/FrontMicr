using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Enum;
using BpmBaseApi.Shared.Models.Camunda;
using BpmBaseApi.Shared.Models.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using System.Text.Encodings.Web;
using System.Text.Json;
using System.Text.Json.Nodes;
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
            // --- 1) Ваша старая логика без изменений ---
            var process = await unitOfWork.ProcessRepository
                .GetByFilterAsync(cancellationToken, p => p.ProcessCode == command.ProcessCode)
                ?? throw new HandlerException(
                    $"Процесс с кодом {command.ProcessCode} не найден",
                    ErrorCodesEnum.Business);

            var requestNumber = await helperService
                .GenerateRequestNumberAsync(command.ProcessCode, cancellationToken);

            var options = new JsonSerializerOptions
            {
                Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping
            };

            // сериализуем входной payload в строку
            var payloadJson = JsonSerializer.Serialize(command.Payload, options);

            var regData = payloadReader.ReadSection<RegData>(command.Payload, "regData");
            var processDataDto = payloadReader.ReadSection<ProcessDataDto>(command.Payload, "processData");

            var processDataCreatedEvent = new ProcessDataCreatedEvent
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
                Title = processDataDto.DocumentTitle
            };

            await unitOfWork.ProcessDataRepository
                .RaiseEvent(processDataCreatedEvent, cancellationToken);

            var camundaVariables = new Dictionary<string, object>
            {
                { "processGuid", processDataCreatedEvent.EntityId.ToString() }
            };

            var response = await camundaService.CamundaStartProcess(
                new CamundaStartProcessRequest { processCode = command.ProcessCode, variables = camundaVariables });

            await unitOfWork
                .ProcessDataRepository
                .RaiseEvent
                (
                    new ProcessDataProcessInstanseIdChangedEvent()
                    {
                        EntityId = processDataCreatedEvent.EntityId,
                        ProcessInstanceId = response
                    },
                    cancellationToken
                );

            // --- 2) NEW: Парсим из готовой строки payloadJson массив files вручную ---
            var root = JsonNode.Parse(payloadJson)!.AsObject();
            var filesArray = root["files"] as JsonArray;

            if (filesArray != null)
            {
                foreach (var node in filesArray.OfType<JsonObject>())
                {
                    // извлекаем fileName и fileType
                    var fileIdStr = node["fileId"]?.GetValue<string>();   // <-- NEW
                    var fileName = node["fileName"]?.GetValue<string>();
                    var fileType = node["fileType"]?.GetValue<string>();

                    if (string.IsNullOrWhiteSpace(fileName) || string.IsNullOrWhiteSpace(fileType))
                    {
                        throw new HandlerException(
                            "Каждый объект в \"files\" должен иметь непустые fileName и fileType",
                            ErrorCodesEnum.Business);
                    }
                    // Жёсткая проверка, что fileId — корректный GUID
                    if (string.IsNullOrWhiteSpace(fileIdStr) || !Guid.TryParse(fileIdStr, out var fileId))
                        throw new HandlerException("Каждый объект в \"files\" должен иметь корректный GUID в поле fileId", ErrorCodesEnum.Business);

                    var fileEvent = new ProcessFileCreatedEvent
                    {
                        EntityId = Guid.NewGuid(),                                // PK → Id записи
                        ProcessDataId = processDataCreatedEvent.EntityId,              // FK на ProcessData
                        FileId   = fileId,                               // <-- NEW
                        FileName = fileName,
                        FileType = fileType
                    };
                    await unitOfWork.ProcessFileRepository
                        .RaiseEvent(fileEvent, cancellationToken);
                }
                // после всех RaiseEvent можно один раз закоммитить,
                // но JournaledGenericRepository.RaiseEvent по умолчанию сразу коммитит
            }
            // --------------------------------------------------------------------

            // --- 3) Старая логика истории задачи и возвращение ответа ---
            var processTaskHistoryCreatedEvent = new ProcessTaskHistoryCreatedEvent
            {
                ProcessDataId = processDataCreatedEvent.EntityId,
                TaskId = processDataCreatedEvent.EntityId,
                Action = ProcessAction.Start.ToString(),
                BlockName = "Регистрационная форма",
                Timestamp = DateTime.Now,
                PayloadJson = payloadJson,
                Comment = "",
                Description = "",
                ProcessCode = command.ProcessCode,
                ProcessName = process.ProcessName,
                RegNumber = requestNumber,
                InitiatorCode = regData.UserCode,
                InitiatorName = regData.UserName,
                Title = processDataDto.DocumentTitle
            };
            await unitOfWork.ProcessTaskHistoryRepository
                .RaiseEvent(processTaskHistoryCreatedEvent, cancellationToken);

            return new BaseResponseDto<StartProcessResponse>
            {
                Data = new StartProcessResponse
                {
                    ProcessGuid = processDataCreatedEvent.EntityId,
                    RegNumber = requestNumber
                }
            };
        }

        /*public async Task<BaseResponseDto<StartProcessResponse>> Handle(StartProcessCommand command, CancellationToken cancellationToken)
        {
            var process = await unitOfWork.ProcessRepository.GetByFilterAsync(cancellationToken, p => p.ProcessCode == command.ProcessCode)
                ?? throw new HandlerException($"Процесс с кодом {command.ProcessCode} не найден", ErrorCodesEnum.Business);

            var currentBlock = await unitOfWork.BlockRepository.GetByIdAsync(cancellationToken, process.StartBlockId!.Value)
                ?? throw new HandlerException("Стартовый шаг не найден", ErrorCodesEnum.Business);

            var requestNumber = await helperService.GenerateRequestNumberAsync(command.ProcessCode, cancellationToken);

            var options = new JsonSerializerOptions
            {
                Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping
            };

            var payloadJson = JsonSerializer.Serialize(command.Payload, options);
            var regData = payloadReader.ReadSection<RegData>(command.Payload, "regData");
            var formData = payloadReader.ReadSection<FormDataDto>(command.Payload, "formData");

            var processDataCreatedEvent = new ProcessDataCreatedEvent
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
                Title = formData.Name
            };

            await unitOfWork.ProcessDataRepository.RaiseEvent(processDataCreatedEvent, cancellationToken);

            

            
            var sysData = payloadReader.ReadSection<SysData>(command.Payload, "sysData");
            var executor = payloadReader.ReadSection<ExecutorDto>(command.Payload, "formData", "executorTabNumberNavigation");
            

            var departments = payloadReader.ReadDepartment(command.Payload);
            var approvers = payloadReader.ReadApprover(command.Payload);
            
            if (command.ProcessCode== "Activity")
            {
                if (executor.UserCode == regData.UserCode)
                {
                    currentBlock = await unitOfWork.BlockRepository.GetByFilterAsync(cancellationToken,
                        b => b.ProcessId == process.Id && b.Id == currentBlock.NextBlockId)
                        ?? throw new HandlerException("Подробности следующего блока не найдены", ErrorCodesEnum.Business);
                }
                else
                {
                    approvers = new List<ApproverDto>();
                    approvers.Add(new ApproverDto
                    {
                        UserCode = executor.UserCode,
                        UserName = executor.UserName
                    });
                }

                var taskParentCreatedEvent = new ProcessTaskCreatedEvent();
                if (approvers.Count > 1)
                {
                    taskParentCreatedEvent = new ProcessTaskCreatedEvent
                    {
                        ProcessDataId = processDataCreatedEvent.EntityId,
                        BlockId = currentBlock.Id,
                        BlockCode = currentBlock.BlockCode,
                        BlockName = currentBlock.BlockName,
                        AssigneeCode = "system",
                        AssigneeName = "system",
                        Status = "Waiting",
                        ProcessCode = process.ProcessCode,
                        ProcessName = process.ProcessName,
                        RegNumber = requestNumber,
                        InitiatorCode = regData.UserCode,
                        InitiatorName = regData.UserName,
                        Title = formData.Name
                    };
                    await unitOfWork.ProcessTaskRepository.RaiseEvent(taskParentCreatedEvent, cancellationToken);
                }
                foreach (var approverDto in approvers)
                {
                    var taskCreatedEvent = new ProcessTaskCreatedEvent
                    {
                        ProcessDataId = processDataCreatedEvent.EntityId,
                        BlockId = currentBlock.Id,
                        BlockCode = currentBlock.BlockCode,
                        BlockName = currentBlock.BlockName,
                        Status = "Pending",
                        PayloadJson = payloadJson,
                        ProcessCode = command.ProcessCode,
                        ProcessName = process.ProcessName,
                        RegNumber = requestNumber,
                        ParentTaskId = taskParentCreatedEvent.EntityId == Guid.Empty ? (Guid?)null : taskParentCreatedEvent.EntityId,
                        InitiatorCode = regData.UserCode,
                        InitiatorName = regData.UserName,
                        AssigneeCode = approverDto.UserCode,
                        AssigneeName = approverDto.UserName,
                        Title = formData.Name
                    };
                    await unitOfWork.ProcessTaskRepository.RaiseEvent(taskCreatedEvent, cancellationToken);
                    await helperService.RefreshUserTaskCacheAsync(approverDto.UserCode, cancellationToken);
                }


                var processTaskHistoryCreatedEvent = new ProcessTaskHistoryCreatedEvent
                {
                    ProcessDataId = processDataCreatedEvent.EntityId,
                    TaskId = processDataCreatedEvent.EntityId, // Возможно стоит заменить на ID задачи, если будет доступен
                    Action = ProcessAction.Start.ToString(),
                    BlockName = "Регистрационная форма",
                    Timestamp = DateTime.Now,
                    PayloadJson = payloadJson,
                    Comment = sysData?.Comment,
                    Description = sysData?.Comment,
                    ProcessCode = command.ProcessCode,
                    ProcessName = process.ProcessName,
                    RegNumber = requestNumber,
                    InitiatorCode = regData.UserCode,
                    InitiatorName = regData.UserName,
                    Title = formData.Name
                };

                await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(processTaskHistoryCreatedEvent, cancellationToken);
            }

            await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataBlockChangedEvent
            {
                EntityId = processDataCreatedEvent.EntityId,
                BlockCode = currentBlock.BlockCode,
                BlockName = currentBlock.BlockName 
            }, cancellationToken);

            return new BaseResponseDto<StartProcessResponse>
            {
                Data = new StartProcessResponse
                {
                    ProcessGuid = processDataCreatedEvent.EntityId,
                    RegNumber = requestNumber,
                    BlockCode = currentBlock.BlockCode,
                    BlockName = currentBlock.BlockName
                }
            };
        }*/
    }
}

using System.Text.Json;
using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Implementations;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Enum;
using BpmBaseApi.Shared.Models.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using static BpmBaseApi.Shared.Models.Process.CommonData;
using static Microsoft.EntityFrameworkCore.DbLoggerCategory.Database;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    public class SetProcessStageCommandHandler(
        IUnitOfWork unitOfWork,
        IProcessTaskService processTaskService
        ) : IRequestHandler<SetProcessStageCommand, BaseResponseDto<SetProcessStageResponse>>
    {
        public async Task<BaseResponseDto<SetProcessStageResponse>> Handle(SetProcessStageCommand request, CancellationToken cancellationToken)
        {
            var processData = await unitOfWork
                .ProcessDataRepository
                .GetByFilterAsync(
                    cancellationToken,
                    a => a.Id == request.ProcessGuid
                );

            if (processData is null)
                return new BaseResponseDto<SetProcessStageResponse> { Data = new SetProcessStageResponse() { Status = "Ok" } };

            var processStageInfo = await unitOfWork.RefProcessStageRepository
                .GetByFilterAsync(
                    cancellationToken,
                    a => a.Code == request.Stage.ToString()
                );

            await unitOfWork
                .ProcessDataRepository
                .RaiseEvent(new ProcessDataBlockChangedEvent
                {
                    EntityId = processData.Id,
                    BlockCode = processStageInfo.Code,
                    BlockName = processStageInfo.Name,
                }, cancellationToken);

            var payloadDict = JsonSerializer.Deserialize<Dictionary<string, object>>(processData.PayloadJson);

            string key = request.Stage switch
            {
                ProcessStage.Approval => "approvers",
                ProcessStage.Signing => "signer",
                ProcessStage.Rework => "initiator",
                ProcessStage.Execution => "recipients",
                ProcessStage.ExecutionCheck => "initiator",
                _ => throw new InvalidOperationException($"Unsupported process stage: {request.Stage}")
            };

            var recipients = ExtractFromPayloadFlexible<UserDataDto>(payloadDict, key) ?? new();

            // 1) читаем тип согласования из processData.approvalTypeCode
            bool isSequential = ReadIsSequentialFromProcessData(payloadDict);

            // 2) родительская system‑задача — как раньше
            var systemTask = await processTaskService.CreateSystemTaskAsync(
                processStageInfo.Code, processStageInfo.Name, processData, cancellationToken);

            // 3) Если Approval и режим Sequentially — создаём детей Pending/Waiting и пишем Order
            if (request.Stage == ProcessStage.Approval && isSequential)
            {
                var ordered = recipients
                    .Select((r, idx) => new { Item = r, Index = idx })
                    .OrderBy(x => x.Item.Order ?? int.MaxValue)
                    .ThenBy(x => x.Item.UserCode ?? "")
                    .ThenBy(x => x.Index)
                    .Select(x => x.Item)
                    .ToList();

                int seq = 1;
                bool isFirst = true;
                foreach (var r in ordered)
                {
                    var assignee = r.UserCode; // loginAD -> UserCode
                    if (string.IsNullOrWhiteSpace(assignee)) continue;

                    await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
                    {
                        EntityId       = Guid.NewGuid(),
                        ProcessDataId  = processData.Id,
                        ParentTaskId   = systemTask.EntityId,

                        ProcessCode    = processData.ProcessCode,
                        ProcessName    = processData.ProcessName ?? "",
                        RegNumber      = processData.RegNumber ?? "",
                        InitiatorCode  = processData.InitiatorCode ?? "",
                        InitiatorName  = processData.InitiatorName ?? "",
                        Title          = processData.Title ?? "",

                        AssigneeCode   = assignee.ToLowerInvariant(),
                        AssigneeName   = r.UserName ?? "",
                        BlockCode      = processStageInfo.Code,
                        BlockName      = processStageInfo.Name,
                        Status         = isFirst ? "Pending" : "Waiting",
                        Order          = r.Order ?? seq
                    }, cancellationToken);

                    isFirst = false;
                    seq++;
                }
            }
            else
            {
                // 4) Иначе — параллельно, как было. Метод внутри уже пишет Order
                await processTaskService.CreateTasksNewAsync(
                    processStageInfo.Code, processStageInfo.Name,
                    recipients, processData, systemTask.EntityId, cancellationToken);
            }

            return new BaseResponseDto<SetProcessStageResponse> { Data = new SetProcessStageResponse() { Status = "Ok" } };
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

        private List<T> ExtractFromPayloadFlexible<T>(Dictionary<string, object> payload, string key)
        {
            if (!payload.TryGetValue(key, out var rawValue) || rawValue == null)
                return new();

            var json = JsonSerializer.Serialize(rawValue);

            try
            {
                // Пытаемся десериализовать как список
                return JsonSerializer.Deserialize<List<T>>(json) ?? new();
            }
            catch (JsonException)
            {
                try
                {
                    // Если не список — пробуем как одиночный объект и оборачиваем его в список
                    var singleItem = JsonSerializer.Deserialize<T>(json);
                    return singleItem != null ? new List<T> { singleItem } : new();
                }
                catch (JsonException ex)
                {
                    throw new JsonException($"Не удалось десериализовать поле '{key}' как {typeof(T)} или List<{typeof(T)}>", ex);
                }
            }
        }
        
        private static bool TryReadSequenceFlag(Dictionary<string, object>? payload)
        {
            if (payload == null) return false;
            try
            {
                if (!payload.TryGetValue("sysInfo", out var sysInfoRaw) || sysInfoRaw == null)
                    return false;

                var json = JsonSerializer.Serialize(sysInfoRaw);
                var sysInfo = JsonSerializer.Deserialize<Dictionary<string, object>>(json);

                if (sysInfo != null && sysInfo.TryGetValue("sequence", out var seqRaw))
                {
                    var seqJson = JsonSerializer.Serialize(seqRaw);
                    var seq = JsonSerializer.Deserialize<bool?>(seqJson);
                    return seq == true;
                }
            }
            catch { /* игнор, считаем false */ }

            return false;
        }

        /// <summary>
        /// true, если processData.approvalTypeCode == "Sequentially" (без учёта регистра)
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
            catch { /* считаем Parallel */ }

            return false;
        }
    }
}
using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Enum;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using System.Text.Json;
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

            // Если sequential — закрываем текущего, поднимаем следующего Waiting -> Pending по Order, затем Created
            if (isSequential)
            {
                var parent = await unitOfWork.ProcessTaskRepository
                    .GetByIdAsync(cancellationToken, currentTask.ParentTaskId!.Value);

                var siblings = await unitOfWork.ProcessTaskRepository
                    .GetByFilterListAsync(cancellationToken, t => t.ParentTaskId == parent.Id);

                await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode,
                    cancellationToken);
                await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

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

                    return new BaseResponseDto<SendProcessResponse>
                    {
                        Data = new SendProcessResponse { Success = true }
                    };
                }

                // Очередь исчерпана — ниже пошла твоя старая логика с Camunda/родителем
            }

            var pendingSiblings = await unitOfWork.ProcessTaskRepository.CountAsync(cancellationToken,
                t => t.ParentTaskId == currentTask.ParentTaskId && t.Id != currentTask.Id /*&& t.Status == "Pending"*/);

            if (pendingSiblings > 0)
            {
                await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode,
                    cancellationToken);
                await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

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

                if (!Enum.TryParse<ProcessStage>(parentTask.BlockCode, out var stage))
                {
                    throw new InvalidOperationException($"Unknown process stage: {parentTask.BlockCode}");
                }

                var submitResponse = await camundaService.CamundaSubmitTask(
                    new CamundaSubmitTaskRequest
                    {
                        TaskId = claimedTasksResponse.TaskId,
                        Variables = GetCamundaVariablesForStage(stage)
                    });

                if (!submitResponse.Success)
                {
                    throw new HandlerException($"Ошибка при отправке Msg:{submitResponse.Msg}", ErrorCodesEnum.Camunda);
                }

                await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode,
                    cancellationToken);
                await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
                await processTaskService.FinalizeTaskAsync(parentTask, cancellationToken);

                return new BaseResponseDto<SendProcessResponse>
                {
                    Data = new SendProcessResponse { Success = true }
                };
            }

            await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
            {
                EntityId = parentTask.Id,
                Status = "Pending"
            }, cancellationToken);

            return new BaseResponseDto<SendProcessResponse>
            {
                Data = new SendProcessResponse { Success = true }
            };
        }

        private Dictionary<string, object> GetCamundaVariablesForStage(ProcessStage stage)
        {
            return stage switch
            {
                ProcessStage.Approval => new() { { "agreement", true } },
                ProcessStage.Signing => new() { { "sign", true } },
                _ => new()
            };
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

        /*
         public async Task<BaseResponseDto<SendProcessResponse>> Handle(SendProcessCommand command, CancellationToken cancellationToken)
        {
            var currentTask = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(cancellationToken,
                p => p.Id == command.TaskId &&
        p.Status == "Pending")
                ?? throw new HandlerException("Задача не найдена или уже обработана", ErrorCodesEnum.Business);

        var processData = await unitOfWork.ProcessDataRepository.GetByIdAsync(cancellationToken, currentTask.ProcessDataId)
            ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

        var recipients = ExtractFromPayload<List<UserInfo>>(command.PayloadJson, "recipients") ?? new();
        var comment = ExtractFromPayload<string>(command.PayloadJson, "comment");


            // ===== Делегирование =====
            if (command.Action == ProcessAction.Delegate && recipients.Any())
            {
                await processTaskService.HandleDelegationAsync(currentTask, processData, command, recipients, comment, cancellationToken);
                //await processTaskService.LogHistoryAsync(currentTask, command, "", cancellationToken);
                return processTaskService.SuccessResponse(currentTask.BlockCode, currentTask.BlockName);
            }

            // ===== Переход к следующему блоку =====
            var currentBlock = await processTaskService.FindNextBlockAsync(
                processData.ProcessId,
                currentTask.BlockId,
     rte           command.Action.ToString(),
        command.Condition?.ToString(),
        cancellationToken
            ) ?? throw new HandlerException("Следующий блок не найден", ErrorCodesEnum.Business);

        var nextBlock = await unitOfWork.BlockRepository.GetByFilterAsync(cancellationToken,
            b => b.ProcessId == processData.ProcessId && b.Id == currentBlock.NextBlockId)
            ?? throw new HandlerException("Подробности следующего блока не найдены", ErrorCodesEnum.Business);


        // Завершение текущей подзадачи и проверка родителя
        await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
        await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
            //await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

            recipients = await processTaskService.ResolveRecipientsAsync(processData.ProcessCode, nextBlock.BlockCode, recipients, cancellationToken);

    var blockResult = await processTaskService.TryHandleParentTaskAsync(currentTask, currentBlock, nextBlock, processData, command, comment, recipients, cancellationToken);

            if (blockResult?.Handled == true)
            {
                if (blockResult.BlockCode == "Completion")
                {
                    await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataStatusChangedEvent
                    {
                        EntityId = processData.Id,
                        StatusCode = "Completed",
                        StatusName = "Завершено"
                    }, cancellationToken);
                }
                await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataBlockChangedEvent
                {
                    EntityId = processData.Id,
                    BlockCode = blockResult.BlockCode,
                    BlockName = blockResult.BlockName
                }, cancellationToken);
return processTaskService.SuccessResponse(blockResult.BlockCode, blockResult.BlockName);
            }

            var parentCreatedEvent = await processTaskService.CreateParentIfNeededAsync(recipients, nextBlock, processData, command, comment, cancellationToken);
await processTaskService.CreateTasksAsync(nextBlock.Id, nextBlock.BlockCode, nextBlock.BlockName, recipients, processData, command, parentCreatedEvent?.EntityId, cancellationToken);

await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataBlockChangedEvent
{
    EntityId = processData.Id,
    BlockCode = nextBlock.BlockCode,
    BlockName = nextBlock.BlockName
}, cancellationToken);

return processTaskService.SuccessResponse(nextBlock.BlockCode, nextBlock.BlockName);
        }
         */
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

        private static bool TryReadSequenceFlag(Dictionary<string, object>? payload)
        {
            if (payload == null) return false;
            try
            {
                if (!payload.TryGetValue("sysInfo", out var sysInfoRaw) || sysInfoRaw == null)
                    return false;

                var json = JsonSerializer.Serialize(sysInfoRaw);
                var sysInfo = JsonSerializer.Deserialize<Dictionary<string, object>>(json);

                if (sysInfo != null && sysInfo.TryGetValue("sequence", out var seqRaw))
                {
                    var seqJson = JsonSerializer.Serialize(seqRaw);
                    var seq = JsonSerializer.Deserialize<bool?>(seqJson);
                    return seq == true;
                }
            }
            catch
            {
                /* игнор, считаем false */
            }

            return false;
        }
    }
}



