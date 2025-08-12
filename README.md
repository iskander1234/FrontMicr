{
  "processCode": "Memo",
  "initiatorCode": "m.ilespayev",
  "initiatorName": "Илеспаев",
  "payload": {"regData":{"userCode":"b.shymkentbay","userName":"Шымкентбай Бақытжан Бахтиярұлы","departmentId":"19.100512","departmentName":"Управление разработки пенсионного учета","startdate":"2025-08-05T10:09:46.584Z","regnum":""},"sysInfo":{"userCode":"b.shymkentbay","userName":"Шымкентбай Бақытжан Бахтиярұлы","comment":"comment","action":"submit","condition":"string"},"initiator":{"id":7820,"name":"Шымкентбай Бақытжан Бахтиярұлы","position":"Главный специалист","login":"b.shymkentbay","statusCode":6,"statusDescription":"Работа","depId":"19.100512","depName":"Управление разработки пенсионного учета","parentDepId":"19.100500","parentDepName":"Департамент цифровизации","isFilial":false,"mail":"b.shymkentbay@enpf.kz","localPhone":"0","mobilePhone":"+7(708) 927-44-98","isManager":false,"managerTabNumber":"4340","disabled":false,"tabNumber":"00ЗП-00292"},"approvers":[{"loginAD":"m.ilespayev","id":611,"name":"Илеспаев Меииржан Анварович","shortName":null,"position":"Заместитель директора департамента","login":"m.ilespayev","statusCode":6,"statusDescription":"Работа","depId":"19.100500","depName":"Департамент цифровизации","parentDepId":"19.100500","parentDepName":"Департамент цифровизации","isFilial":false,"mail":"m.ilespayev@enpf.kz","localPhone":"0","mobilePhone":"+7(702) 171-71-14","isManager":true,"managerTabNumber":"4303","disabled":false,"tabNumber":"00ЗП-00240"},{"loginAD":"a.ysmail","id":1545,"name":"Ысмаил Арғынбек Байдабекұлы","shortName":null,"position":"Главный специалист","login":"a.ysmail","statusCode":6,"statusDescription":"Работа","depId":"19.100508","depName":"Управление разработки фронтальных систем","parentDepId":"19.100500","parentDepName":"Департамент цифровизации","isFilial":false,"mail":"a.ysmail@enpf.kz","localPhone":"0","mobilePhone":"+7(702) 778-53-30","isManager":false,"managerTabNumber":"4340","disabled":false,"tabNumber":"00ЗП-00289"}],"recipients":[{"loginAD":"l.iskender","id":633,"name":"Искендер Лесхан Муратұлы","shortName":null,"position":"Главный специалист","login":"l.iskender","statusCode":6,"statusDescription":"Работа","depId":"19.100509","depName":"Управление разработки Web приложений и сервисов","parentDepId":"19.100500","parentDepName":"Департамент цифровизации","isFilial":false,"mail":"l.iskender@enpf.kz","localPhone":"0","mobilePhone":"+7(707) 517-04-67","isManager":false,"managerTabNumber":"4340","disabled":false,"tabNumber":"00ЗП-00083"},{"loginAD":"i.dosgali","id":414,"name":"Досгали Искандер Досгалиұлы","shortName":null,"position":"Главный специалист","login":"i.dosgali","statusCode":6,"statusDescription":"Работа","depId":"19.100509","depName":"Управление разработки Web приложений и сервисов","parentDepId":"19.100500","parentDepName":"Департамент цифровизации","isFilial":false,"mail":"i.dosgali@enpf.kz","localPhone":"0","mobilePhone":"+7(747) 790-29-49","isManager":false,"managerTabNumber":"4340","disabled":false,"tabNumber":"00ЗП-00275"}],"signer":{"loginAD":"a.aristombekov","id":168,"name":"Аристомбеков Арстан Рамазанулы","shortName":null,"position":"Директор департамента","login":"a.aristombekov","statusCode":5,"statusDescription":"Отпуск основной","depId":"19.100500","depName":"Департамент цифровизации","parentDepId":"19.100500","parentDepName":"Департамент цифровизации","isFilial":false,"mail":"a.aristombekov@enpf.kz","localPhone":"0","mobilePhone":"+7(705) 950-90-65","isManager":true,"managerTabNumber":"4303","disabled":false,"tabNumber":"4340"},"processData":{"documentLang":"","nomenclatureId":"1","documentTitle":"тема документа","grif":"string","pageCount":"5","signType":"string","allGroup":"string","meGroup":"string","documentBody":"<p><strong>ТЕКС&nbsp;СЗ</strong></p>"},"files": [
    {
        "fileId" : "0ec544b8-6d84-4fd5-93fc-ff22f13053b3",
        "fileName" : "Рисковое событие.docx",
        "fileType" : "scan"
    },
    {
        "fileId" : "8d1b2532-b727-484d-877f-1e7658851a2e",
        "fileName" : "ffh.pdf",
        "fileType" : "scan"
    }
]
}
}
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

    var regData        = payloadReader.ReadSection<RegData>(command.Payload, "regData");
    var processDataDto = payloadReader.ReadSection<ProcessDataDto>(command.Payload, "processData");

    var response = await camundaService.CamundaStartProcess(
        new CamundaStartProcessRequest { processCode = command.ProcessCode });

    var processDataCreatedEvent = new ProcessDataCreatedEvent
    {
        ProcessId         = process.Id,
        ProcessCode       = process.ProcessCode,
        ProcessName       = process.ProcessName,
        RegNumber         = requestNumber,
        InitiatorCode     = regData.UserCode,
        InitiatorName     = regData.UserName,
        StatusCode        = "Started",
        StatusName        = "В работе",
        PayloadJson       = payloadJson,
        Title             = processDataDto.DocumentTitle,
        ProcessInstanceId = response
    };
    await unitOfWork.ProcessDataRepository
        .RaiseEvent(processDataCreatedEvent, cancellationToken);

    // --- 2) NEW: Парсим из готовой строки payloadJson массив files вручную ---
    var root = JsonNode.Parse(payloadJson)!.AsObject();
    var filesArray = root["files"] as JsonArray;

    if (filesArray != null)
    {
        foreach (var node in filesArray.OfType<JsonObject>())
        {
            // извлекаем fileName и fileType
            var fileName = node["fileName"]?.GetValue<string>();
            var fileType = node["fileType"]?.GetValue<string>();

            if (string.IsNullOrWhiteSpace(fileName) || string.IsNullOrWhiteSpace(fileType))
            {
                throw new HandlerException(
                    "Каждый объект в \"files\" должен иметь непустые fileName и fileType",
                    ErrorCodesEnum.Business);
            }

            var fileEvent = new ProcessFileCreatedEvent
            {
                EntityId      = Guid.NewGuid(),                                // PK → Id записи
                ProcessDataId = processDataCreatedEvent.EntityId,              // FK на ProcessData
                FileName      = fileName,
                FileType      = fileType
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
        ProcessDataId    = processDataCreatedEvent.EntityId,
        TaskId           = processDataCreatedEvent.EntityId,
        Action           = ProcessAction.Start.ToString(),
        BlockName        = "Регистрационная форма",
        Timestamp        = DateTime.Now,
        PayloadJson      = payloadJson,
        Comment          = "",
        Description      = "",
        ProcessCode      = command.ProcessCode,
        ProcessName      = process.ProcessName,
        RegNumber        = requestNumber,
        InitiatorCode    = regData.UserCode,
        InitiatorName    = regData.UserName,
        Title            = processDataDto.DocumentTitle
    };
    await unitOfWork.ProcessTaskHistoryRepository
        .RaiseEvent(processTaskHistoryCreatedEvent, cancellationToken);

    return new BaseResponseDto<StartProcessResponse>
    {
        Data = new StartProcessResponse
        {
            ProcessGuid = processDataCreatedEvent.EntityId,
            RegNumber   = requestNumber
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

}using System.Text.Json;
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

            var systemTask = await processTaskService.CreateSystemTaskAsync(processStageInfo.Code, processStageInfo.Name, processData, cancellationToken);

            await processTaskService.CreateTasksNewAsync(processStageInfo.Code, processStageInfo.Name, recipients, processData, systemTask.EntityId, cancellationToken);

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
using System.Diagnostics;
using System;
using System.Text.Json;
using AutoMapper;
using Microsoft.Extensions.Caching.Memory;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Services.Interfaces;
using SixLabors.ImageSharp.Processing.Processors.Transforms;
using BpmBaseApi.Shared.Models.Process;
using BpmBaseApi.Persistence;
using BpmBaseApi.Services.Implementations;
using static Microsoft.EntityFrameworkCore.DbLoggerCategory.Database;
using System.Threading;
using BpmBaseApi.Services.Camunda;
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
        public async Task<BaseResponseDto<SendProcessResponse>> Handle(SendProcessCommand command, CancellationToken cancellationToken)
        {
            var currentTask = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(cancellationToken,
                p => p.Id == command.TaskId && /*p.AssigneeCode.ToLower() == command.SenderUserCode.ToLower() &&*/ p.Status == "Pending")
                ?? throw new HandlerException("Задача не найдена или уже обработана", ErrorCodesEnum.Business);

            var processData = await unitOfWork.ProcessDataRepository.GetByIdAsync(cancellationToken, currentTask.ProcessDataId)
                ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

            var recipients = ExtractFromPayload<List<UserInfo>>(command.PayloadJson, "recipients") ?? new();
            var comment = ExtractFromPayload<string>(command.PayloadJson, "comment");
            

            // ===== Делегирование =====
            if (command.Action == ProcessAction.Delegate && recipients.Any())
            {
                await processTaskService.HandleDelegationAsync(currentTask, processData, command, recipients, comment, cancellationToken);
                return processTaskService.SuccessResponse(currentTask.BlockCode, currentTask.BlockName);
            }


            
            
           
            var pendingSiblings = await unitOfWork.ProcessTaskRepository.CountAsync(cancellationToken,
                t => t.ParentTaskId == currentTask.ParentTaskId && t.Id != currentTask.Id /*&& t.Status == "Pending"*/);

            if (pendingSiblings > 0)
            {
                await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
                await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

                return new BaseResponseDto<SendProcessResponse>
                {
                    Data = new SendProcessResponse
                    {
                        Success = true
                    }
                };
            }

            var parentTask = await unitOfWork.ProcessTaskRepository.GetByIdAsync(cancellationToken, currentTask.ParentTaskId.Value);

            if (parentTask.AssigneeCode == "system")
            {
                var claimedTasksResponse = await camundaService.CamundaClaimedTasks(processData.ProcessInstanceId, parentTask.BlockCode)
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

                await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
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

