Мне надо убрать  "sysInfo": {
      "userCode": "b.shymkentbay",
      "userName": "Шымкентбай Бақытжан Бахтиярұлы",
      "comment": "comment",
      "action": "submit",
      "condition": "string",
      "sequence": true именно sequence сделать сюда  "processData": {
      "documentTitle": "тема документа",
      "approvalTypeCode": "Parallel", или "Sequentially"
      "approvalTypeName": "Параллельно", или "Последовательно"
      "nomenclatureId": "1",
      "nomenclatureName": "Акт",
      "documentLang": "ru",      
      "confLevelCode": "string",
      "confLevelName": "string",
      "pageCount": "5",
      "allGroups": "string",
      "myGroups": "string",
      "DueDate": "date",
      "documentBody": "<p><strong>ТЕКС&nbsp;СЗ</strong></p>" потом мне надо делать ProcessTask в таблицу добавить Order и  "approvers": [
      {
        "loginAD": "m.ilespayev",
        "id": 611,
        "name": "Илеспаев Меииржан Анварович",
        "shortName": null,
        "position": "Заместитель директора департамента",
        "login": "m.ilespayev",
        "statusCode": 6,
        "statusDescription": "Работа",
        "depId": "19.100500",
        "depName": "Департамент цифровизации",
        "parentDepId": "19.100500",
        "parentDepName": "Департамент цифровизации",
        "isFilial": false,
        "mail": "m.ilespayev@enpf.kz",
        "localPhone": "0",
        "mobilePhone": "+7(702) 171-71-14",
        "isManager": true,
        "managerTabNumber": "4303",
        "disabled": false,
        "tabNumber": "00ЗП-00240",
        "order" :1
      }, order заисывать в ProcessTask тоже 
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
                //throw new HandlerException($"Процесс с идентификатором {request.InstanceId} не найден");
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

            // var systemTask = await processTaskService.CreateSystemTaskAsync(processStageInfo.Code, processStageInfo.Name, processData, cancellationToken);
            //
            // await processTaskService.CreateTasksNewAsync(processStageInfo.Code, processStageInfo.Name, recipients, processData, systemTask.EntityId, cancellationToken);
           // var payloadDict = JsonSerializer.Deserialize<Dictionary<string, object>>(processData.PayloadJson);

// 1) читаем флаг последовательности
            bool isSequential = TryReadSequenceFlag(payloadDict);

// 2) родительская system‑задача — как раньше
            var systemTask = await processTaskService.CreateSystemTaskAsync(
                processStageInfo.Code, processStageInfo.Name, processData, cancellationToken);

// 3) если это этап Approval и включён последовательный режим — создаём детей с Pending/Waiting
            if (request.Stage == ProcessStage.Approval && isSequential)
            {
                var ordered = recipients
                    .Select((r, idx) => new { Item = r, Index = idx })
                    .OrderBy(x => x.Item.Order ?? int.MaxValue)
                    .ThenBy(x => x.Item.UserCode ?? "")   // стабилизируем при одинаковом Order
                    .ThenBy(x => x.Index)                 // финальная стабилизация по позиции в массиве
                    .Select(x => x.Item)
                    .ToList();

                bool isFirst = true;
                foreach (var r in ordered)
                {
                    var assignee = r.UserCode;            // у тебя UserCode ← loginAD
                    if (string.IsNullOrWhiteSpace(assignee)) continue;

                    await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
                    {
                        EntityId       = Guid.NewGuid(),
                        ProcessDataId  = processData.Id,
                        ParentTaskId   = systemTask.EntityId,

                        // ВАЖНО: эти поля обязательны в БД
                        ProcessCode    = processData.ProcessCode,         // NOT NULL
                        ProcessName    = processData.ProcessName ?? "",   // на всякий случай
                        RegNumber      = processData.RegNumber ?? "",
                        InitiatorCode  = processData.InitiatorCode ?? "",
                        InitiatorName  = processData.InitiatorName ?? "",
                        Title          = processData.Title ?? "",

                        AssigneeCode   = assignee.ToLowerInvariant(),
                        AssigneeName   = r.UserName ?? "",
                        BlockCode      = processStageInfo.Code,
                        BlockName      = processStageInfo.Name,
                        Status         = isFirst ? "Pending" : "Waiting"
                    }, cancellationToken);


                    isFirst = false;
                }
            }
            else
            {
                // 4) во всех остальных случаях (включая Approval с sequence=false) — старое поведение (параллельно)
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
            // 1) Берём задачу БЕЗ фильтра по статусу (чтобы поймать случай Waiting и вернуть 400 с подсказкой)
            var currentTask = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                cancellationToken,
                p => p.Id == command.TaskId
            ) ?? throw new HandlerException("Задача не найдена", ErrorCodesEnum.Business);

            var processData =
                await unitOfWork.ProcessDataRepository.GetByIdAsync(cancellationToken, currentTask.ProcessDataId)
                ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

            // 2) Понимаем — последовательный ли режим (флаг из сохранённого payload)
            bool isSequential = false;
            var pdPayload = JsonSerializer.Deserialize<Dictionary<string, object>>(processData.PayloadJson);
            try
            {
                isSequential = TryReadSequenceFlag(pdPayload);
            }
            catch
            {
                isSequential = false;
            }

            // 3) Если последовательный режим и задача НЕ Pending → вернуть 400,
            //    указав кто должен отправить (текущий обладатель Pending)
            if (isSequential && !string.Equals(currentTask.Status, "Pending", StringComparison.OrdinalIgnoreCase))
            {
                // ищем активного Pending по тому же родителю
                ProcessTaskEntity? activePending = null;
                if (currentTask.ParentTaskId != null)
                {
                    activePending = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                        cancellationToken,
                        t => t.ParentTaskId == currentTask.ParentTaskId && t.Status == "Pending"
                    );
                }

                // запасной поиск, если родителя нет (или не нашли)
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
                    ErrorCodesEnum.Business //  это маппится в HTTP 400 ? 
                );
            }

            // 4) Дальше — старая проверка «только Pending» (для параллельного режима как было)
            if (!string.Equals(currentTask.Status, "Pending", StringComparison.OrdinalIgnoreCase))
                throw new HandlerException("Задача не найдена или уже обработана", ErrorCodesEnum.Business);

            //  ДАЛЕЕ ВЕСЬ ТВОЙ ИСХОДНЫЙ КОД БЕЗ ИЗМЕНЕНИЙ 

            var recipients = ExtractFromPayload<List<UserInfo>>(command.PayloadJson, "recipients") ?? new();
            var comment = ExtractFromPayload<string>(command.PayloadJson, "comment");

            // Делегирование
            if (command.Action == ProcessAction.Delegate && recipients.Any())
            {
                await processTaskService.HandleDelegationAsync(currentTask, processData, command, recipients, comment,
                    cancellationToken);
                return processTaskService.SuccessResponse(currentTask.BlockCode, currentTask.BlockName);
            }

            // Определяем, последовательный ли режим (флажок берём из сохранённого PayloadJson)
            // 0) понять режим из сохранённого payload
            //bool isSequential = false;
            //var pdPayload = JsonSerializer.Deserialize<Dictionary<string, object>>(processData.PayloadJson);
            try
            {
                isSequential = TryReadSequenceFlag(pdPayload);
            }
            catch
            {
                isSequential = false;
            }

            if (isSequential)
            {
                // 1) родительская системная задача
                var parent = await unitOfWork.ProcessTaskRepository
                    .GetByIdAsync(cancellationToken, currentTask.ParentTaskId!.Value);

                // 2) ВСЕ дочерние задачи под тем же родителем (используем GetByFilterListAsync!)
                var siblings = await unitOfWork.ProcessTaskRepository
                    .GetByFilterListAsync(cancellationToken, t => t.ParentTaskId == parent.Id);

                // 3) лог + закрываем текущего
                await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode,
                    cancellationToken);
                await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

                // 4) найти следующего Waiting и сделать его Pending
                var next = siblings
                    .Where(t => t.Status == "Waiting")
                    .OrderBy(t => t.Created) // порядок соответствует порядку создания в SetProcessStage
                    .FirstOrDefault();

                if (next != null)
                {
                    await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
                    {
                        EntityId = next.Id,
                        Status = "Pending"
                    }, cancellationToken);

                    // очередь продолжается — Camunda/родителя пока не трогаем
                    return new BaseResponseDto<SendProcessResponse>
                    {
                        Data = new SendProcessResponse { Success = true }
                    };
                }

                // если дошли сюда — очередь исчерпана; ниже пойдет старая логика (Camunda/parent)
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
                    Data = new SendProcessResponse
                    {
                        Success = true
                    }
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
                    Data = new SendProcessResponse
                    {
                        Success = true
                    }
                };
            }

            await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
            {
                EntityId = parentTask.Id,
                Status = "Pending"
            }, cancellationToken);


            return new BaseResponseDto<SendProcessResponse>
            {
                Data = new SendProcessResponse
                {
                    Success = true
                }
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
                command.Action.ToString(),
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
            catch { /* игнор, считаем false */ }

            return false;
        }
    }
}
    

