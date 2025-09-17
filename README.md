using System;
using System.Collections.Generic;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Shared.Commands.Process
{
    /// <summary>
    /// Отправить на ознакомление.
    /// Ожидает в PayloadJson блок:
    /// {
    ///   "acquaintance": {
    ///     "message": "Просьба ознакомиться",
    ///     "recipients": [{ "userCode": "...", "userName": "..." }, ...]
    ///   }
    /// }
    /// </summary>
    public record SendAcquaintanceCommand(
        Guid ProcessDataId,
        Dictionary<string, object>? PayloadJson
    ) : IRequest<BaseResponseDto<SendProcessResponse>>;

    /// <summary>Пометить «Ознакомлен»</summary>
    public record AcknowledgeAcquaintanceCommand(
        Guid TaskId
    ) : IRequest<BaseResponseDto<SendProcessResponse>>;
}


namespace BpmBaseApi.Shared.Enum
{
    public enum ProcessAction
    {
        Submit,
        Delegate,
        Cancel,
        Complete,
        Reject,
        Waiting,
        Start,

        /// <summary>Отправлено на ознакомление</summary>
        Acquaintance
    }
}



using System;
using System.Collections.Generic;
using System.Linq;
using System.Text.Json;
using System.Text.Json.Nodes;
using System.Threading;
using System.Threading.Tasks;
using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Enum;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    // ====== SEND ======
    public class SendAcquaintanceCommandHandler(
        IUnitOfWork unitOfWork,
        IProcessTaskService processTaskService
    ) : IRequestHandler<SendAcquaintanceCommand, BaseResponseDto<SendProcessResponse>>
    {
        private const string AcquaintanceCode = "Acquaintance";

        public async Task<BaseResponseDto<SendProcessResponse>> Handle(SendAcquaintanceCommand cmd, CancellationToken ct)
        {
            var pd = await unitOfWork.ProcessDataRepository.GetByIdAsync(ct, cmd.ProcessDataId)
                     ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

            var (recipients, message) = TryReadFromPayload(cmd.PayloadJson);
            if (recipients.Count == 0)
                throw new HandlerException("В payload.acquaintance.recipients не найден ни один получатель", ErrorCodesEnum.Business);

            // Идемпотентность: не создаём Pending-дубликаты по тем же UserCode
            var existingPending = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
                ct,
                t => t.ProcessDataId == pd.Id
                     && t.BlockCode == AcquaintanceCode
                     && t.Status == "Pending"
                     && !string.IsNullOrEmpty(t.AssigneeCode));

            var alreadyCodes = existingPending
                .Select(t => t.AssigneeCode!)
                .ToHashSet(StringComparer.OrdinalIgnoreCase);

            var toCreate = recipients.Where(r => !alreadyCodes.Contains(r.UserCode)).ToList();
            if (toCreate.Count == 0)
            {
                return new BaseResponseDto<SendProcessResponse>
                {
                    Data = new SendProcessResponse { Success = true }
                };
            }

            // Родитель-группа (скрытая)
            var parentId = Guid.NewGuid();
            await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
            {
                EntityId      = parentId,
                ProcessDataId = pd.Id,
                BlockCode     = AcquaintanceCode,
                BlockName     = "Ознакомление",
                AssigneeCode  = "system",
                Status        = "Hidden",
                Comment       = message
            }, ct);

            // Дочерние Pending-задачи адресатам
            foreach (var r in toCreate)
            {
                await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
                {
                    EntityId      = Guid.NewGuid(),
                    ParentTaskId  = parentId,
                    ProcessDataId = pd.Id,
                    BlockCode     = AcquaintanceCode,
                    BlockName     = "Ознакомление",
                    AssigneeCode  = r.UserCode,
                    AssigneeName  = r.UserName,
                    Status        = "Pending",
                    Comment       = message
                }, ct);

                await processTaskService.RefreshUserTaskCacheAsync(r.UserCode, ct);
            }

            // История
            await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(new ProcessTaskHistoryCreatedEvent
            {
                ProcessDataId = pd.Id,
                TaskId        = parentId,
                BlockCode     = AcquaintanceCode,
                BlockName     = "Ознакомление",
                Action        = ProcessAction.Acquaintance.ToString(),
                ActionName    = "Отправлено на ознакомление",
                Timestamp     = DateTime.Now,
                Description   = string.Join(", ", toCreate.Select(x => $"{x.UserName}({x.UserCode})"))
            }, ct);

            return new BaseResponseDto<SendProcessResponse>
            {
                Data = new SendProcessResponse { Success = true }
            };
        }

        private static (List<UserInfo> Recipients, string? Message) TryReadFromPayload(Dictionary<string, object>? payload)
        {
            var result = new List<UserInfo>();
            string? message = null;

            if (payload is null || payload.Count == 0) return (result, null);

            try
            {
                var json = JsonSerializer.Serialize(payload);
                var root = JsonNode.Parse(json)!.AsObject();
                var acq  = root["acquaintance"] as JsonObject;
                if (acq is null) return (result, null);

                message = acq["message"]?.GetValue<string>();

                if (acq["recipients"] is JsonArray arr)
                {
                    foreach (var n in arr.OfType<JsonObject>())
                    {
                        var code = n["userCode"]?.GetValue<string>()?.Trim();
                        if (string.IsNullOrWhiteSpace(code)) continue;
                        var name = n["userName"]?.GetValue<string>();
                        result.Add(new UserInfo { UserCode = code!, UserName = name });
                    }
                }
            }
            catch
            {
                // игнорируем кривой JSON — вернём пустой список
            }

            // dedupe по UserCode
            result = result
                .GroupBy(x => x.UserCode, StringComparer.OrdinalIgnoreCase)
                .Select(g => g.First())
                .ToList();

            return (result, message);
        }
    }

    // ====== ACK ======
    public class AcknowledgeAcquaintanceCommandHandler(
        IUnitOfWork unitOfWork,
        IProcessTaskService processTaskService
    ) : IRequestHandler<AcknowledgeAcquaintanceCommand, BaseResponseDto<SendProcessResponse>>
    {
        private const string AcquaintanceCode = "Acquaintance";

        public async Task<BaseResponseDto<SendProcessResponse>> Handle(AcknowledgeAcquaintanceCommand cmd, CancellationToken ct)
        {
            var task = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(ct, t => t.Id == cmd.TaskId)
                       ?? throw new HandlerException("Задача не найдена", ErrorCodesEnum.Business);

            if (!string.Equals(task.BlockCode, AcquaintanceCode, StringComparison.OrdinalIgnoreCase))
                throw new HandlerException("Эта задача не является задачею ознакомления", ErrorCodesEnum.Business);

            // идемпотентность — если не Pending, просто успех
            if (!string.Equals(task.Status, "Pending", StringComparison.OrdinalIgnoreCase))
                return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };

            // История
            await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(new ProcessTaskHistoryCreatedEvent
            {
                ProcessDataId = task.ProcessDataId,
                TaskId        = task.Id,
                BlockCode     = task.BlockCode,
                BlockName     = task.BlockName,
                Action        = "Acknowledge",
                ActionName    = "Ознакомлен",
                Timestamp     = DateTime.Now,
                AssigneeCode  = task.AssigneeCode,
                AssigneeName  = task.AssigneeName,
                Description   = "Пользователь подтвердил ознакомление"
            }, ct);

            // Финализируем (уберём из инбокса)
            await processTaskService.FinalizeTaskAsync(task, ct);
            if (!string.IsNullOrWhiteSpace(task.AssigneeCode))
                await processTaskService.RefreshUserTaskCacheAsync(task.AssigneeCode, ct);

            // Закрыть родителя, если больше никто не Pending
            if (task.ParentTaskId.HasValue)
            {
                var stillPending = await unitOfWork.ProcessTaskRepository.CountAsync(
                    ct, t => t.ParentTaskId == task.ParentTaskId && t.Status == "Pending");

                if (stillPending == 0)
                {
                    var parent = await unitOfWork.ProcessTaskRepository.GetByIdAsync(ct, task.ParentTaskId.Value);
                    if (parent is not null)
                        await processTaskService.FinalizeTaskAsync(parent, ct);
                }
            }

            return new BaseResponseDto<SendProcessResponse>
            {
                Data = new SendProcessResponse { Success = true }
            };
        }
    }
}




using System.Threading;
using System.Threading.Tasks;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using Microsoft.AspNetCore.Mvc;
using Swashbuckle.AspNetCore.Annotations;

namespace BpmBaseApi.Api.Controllers
{
    [ApiController]
    [Route("api/process")]
    public class ProcessAcquaintanceController : ControllerBase
    {
        private readonly IMediator _mediator;
        public ProcessAcquaintanceController(IMediator mediator) => _mediator = mediator;

        /// <summary>Отправить на ознакомление</summary>
        [HttpPost("acquaintance/send")]
        [SwaggerResponse(StatusCodes.Status200OK, "OK", typeof(BaseResponseDto<SendProcessResponse>))]
        public async Task<IActionResult> SendAsync([FromBody] SendAcquaintanceCommand cmd, CancellationToken ct)
        {
            var result = await _mediator.Send(cmd, ct);
            return Ok(result);
        }

        /// <summary>Пометить «Ознакомлен»</summary>
        [HttpPost("acquaintance/ack")]
        [SwaggerResponse(StatusCodes.Status200OK, "OK", typeof(BaseResponseDto<SendProcessResponse>))]
        public async Task<IActionResult> AcknowledgeAsync([FromBody] AcknowledgeAcquaintanceCommand cmd, CancellationToken ct)
        {
            var result = await _mediator.Send(cmd, ct);
            return Ok(result);
        }
    }
}



