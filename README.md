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


            
            // NEW: последовательный режим?
var pdPayload = JsonSerializer.Deserialize<Dictionary<string, object>>(processData.PayloadJson);
bool sequential = IsSequential(pdPayload);

if (sequential)
{
    // 1) Запишем историю и закроем текущую
    await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
    await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

    // 2) Ищем следующую Waiting у того же ParentTaskId
    var siblings = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
        cancellationToken,
        t => t.ParentTaskId == currentTask.ParentTaskId
    );

    var next = siblings
        ?.Where(t => t.Status == "Waiting")
        .OrderBy(t => t.Created)
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

    // 3) Ждущих нет — завершаем родителя (system) и двигаем Camunda (ровно как у тебя уже сделано)
    var parentTaskSeq = await unitOfWork.ProcessTaskRepository
        .GetByIdAsync(cancellationToken, currentTask.ParentTaskId!.Value);

    if (parentTaskSeq.AssigneeCode == "system")
    {
        var claimed = await camundaService.CamundaClaimedTasks(processData.ProcessInstanceId, parentTaskSeq.BlockCode)
             ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);

        if (!Enum.TryParse<ProcessStage>(parentTaskSeq.BlockCode, out var stageSeq))
            throw new InvalidOperationException($"Unknown process stage: {parentTaskSeq.BlockCode}");

        var submit = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
        {
            TaskId = claimed.TaskId,
            Variables = GetCamundaVariablesForStage(stageSeq)
        });

        if (!submit.Success)
            throw new HandlerException($"Ошибка при отправке Msg:{submit.Msg}", ErrorCodesEnum.Camunda);

        await processTaskService.FinalizeTaskAsync(parentTaskSeq, cancellationToken);
    }

    return new BaseResponseDto<SendProcessResponse>
    {
        Data = new SendProcessResponse { Success = true }
    };
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

    private static bool IsSequential(Dictionary<string, object>? payload)
    {
        if (payload == null) return false;
        if (!payload.TryGetValue("sysInfo", out var sysRaw) || sysRaw is null) return false;

        var json = JsonSerializer.Serialize(sysRaw);
        var sys  = JsonSerializer.Deserialize<Dictionary<string, object>>(json);

        if (sys != null && sys.TryGetValue("sequence", out var v) && v is not null)
        {
            if (v is bool b) return b;
            if (bool.TryParse(v.ToString(), out var parsed)) return parsed;
        }
        return false;
    }


}
