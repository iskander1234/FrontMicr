namespace BpmBaseApi.Shared.Enum
{
    public enum ProcessAction
    {
        /// <summary>Отправить</summary>
        Submit,
        /// <summary>Делегировать</summary>
        Delegate,
        /// <summary>Отменить</summary>
        Cancel,
        /// <summary>Завершить</summary>
        Complete,
        /// <summary>Отклонить</summary>
        Reject,
        /// <summary>Ожидание</summary>
        Waiting,
        /// <summary>Запуск</summary>
        Start,

        // --- NEW ---
        /// <summary>Отправить на ознакомление</summary>
        Acquaintance,

        /// <summary>Подтвердить ознакомление</summary>
        Acknowledge,

        /// <summary>Алиас для совместимости с опечаткой. Не используйте.</summary>
        [System.Obsolete("Use ProcessAction.Acquaintance")]
        Acquintance = Acquaintance
    }
}


using System;
using System.Collections.Generic;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Shared.Commands.Process
{
    /// <summary>Отправить на ознакомление</summary>
    public record SendAcquaintanceCommand(
        Guid ProcessDataId,
        List<UserInfo> Recipients,
        string? Message
    ) : IRequest<BaseResponseDto<SendProcessResponse>>;

    /// <summary>Пометить «Ознакомлен»</summary>
    public record AcknowledgeAcquaintanceCommand(
        Guid TaskId
    ) : IRequest<BaseResponseDto<SendProcessResponse>>;
}


using System;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Enum;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    public class SendAcquaintanceCommandHandler(
        IUnitOfWork unitOfWork,
        IProcessTaskService processTaskService
    ) : IRequestHandler<SendAcquaintanceCommand, BaseResponseDto<SendProcessResponse>>
    {
        private const string AcquaintanceBlock = "Acquaintance";

        public async Task<BaseResponseDto<SendProcessResponse>> Handle(SendAcquaintanceCommand cmd, CancellationToken ct)
        {
            var pd = await unitOfWork.ProcessDataRepository.GetByIdAsync(ct, cmd.ProcessDataId)
                     ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

            var recipients = (cmd.Recipients ?? new())
                .Where(r => !string.IsNullOrWhiteSpace(r.UserCode))
                .GroupBy(r => r.UserCode)
                .Select(g => g.First())
                .ToList();

            if (recipients.Count == 0)
                throw new HandlerException("Не переданы получатели ознакомления", ErrorCodesEnum.Business);

            // Защита от дублей: не создаём Pending ещё раз тому же человеку по этой же заявке
            var alreadyPending = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
                ct,
                t => t.ProcessDataId == pd.Id
                     && t.BlockCode == AcquaintanceBlock
                     && t.Status == "Pending"
                     && !string.IsNullOrEmpty(t.AssigneeCode)
            );
            var alreadySet = alreadyPending
                .Select(t => t.AssigneeCode!)
                .ToHashSet(StringComparer.OrdinalIgnoreCase);

            var toCreate = recipients.Where(r => !alreadySet.Contains(r.UserCode)).ToList();
            if (toCreate.Count == 0)
            {
                // уже всё есть — просто вернём успех (идемпотентность)
                return new BaseResponseDto<SendProcessResponse>
                {
                    Data = new SendProcessResponse { Success = true }
                };
            }

            // (опционально) создаём родителя-группу
            var parentId = Guid.NewGuid();
            await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
            {
                EntityId      = parentId,
                ProcessDataId = pd.Id,
                BlockCode     = AcquaintanceBlock,
                BlockName     = "Ознакомление",
                AssigneeCode  = "system",
                Status        = "Hidden",        // скрытый служебный узел группы
                Comment       = cmd.Message
            }, ct);

            foreach (var r in toCreate)
            {
                await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
                {
                    EntityId      = Guid.NewGuid(),
                    ParentTaskId  = parentId,
                    ProcessDataId = pd.Id,
                    BlockCode     = AcquaintanceBlock,
                    BlockName     = "Ознакомление",
                    AssigneeCode  = r.UserCode,
                    AssigneeName  = r.UserName,
                    Status        = "Pending",
                    Comment       = cmd.Message
                }, ct);

                await processTaskService.RefreshUserTaskCacheAsync(r.UserCode, ct);
            }

            // История
            await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(new ProcessTaskHistoryCreatedEvent
            {
                ProcessDataId = pd.Id,
                TaskId        = parentId,
                BlockCode     = AcquaintanceBlock,
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
    }
}



using System;
using System.Threading;
using System.Threading.Tasks;
using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Enum;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    public class AcknowledgeAcquaintanceCommandHandler(
        IUnitOfWork unitOfWork,
        IProcessTaskService processTaskService
    ) : IRequestHandler<AcknowledgeAcquaintanceCommand, BaseResponseDto<SendProcessResponse>>
    {
        private const string AcquaintanceBlock = "Acquaintance";

        public async Task<BaseResponseDto<SendProcessResponse>> Handle(AcknowledgeAcquaintanceCommand cmd, CancellationToken ct)
        {
            var task = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(ct, t => t.Id == cmd.TaskId)
                       ?? throw new HandlerException("Задача не найдена", ErrorCodesEnum.Business);

            if (!string.Equals(task.BlockCode, AcquaintanceBlock, StringComparison.OrdinalIgnoreCase))
                throw new HandlerException("Эта задача не является задачей ознакомления", ErrorCodesEnum.Business);

            // Идемпотентность: если уже не Pending — возвращаем успех
            if (!string.Equals(task.Status, "Pending", StringComparison.OrdinalIgnoreCase))
                return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };

            // История
            await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(new ProcessTaskHistoryCreatedEvent
            {
                ProcessDataId = task.ProcessDataId,
                TaskId        = task.Id,
                BlockCode     = task.BlockCode,
                BlockName     = task.BlockName,
                Action        = ProcessAction.Acknowledge.ToString(),
                ActionName    = "Ознакомлен",
                Timestamp     = DateTime.Now,
                AssigneeCode  = task.AssigneeCode,
                AssigneeName  = task.AssigneeName,
                Description   = "Пользователь подтвердил ознакомление"
            }, ct);

            // Финализируем подзадачу (убираем из инбокса)
            await processTaskService.FinalizeTaskAsync(task, ct);
            if (!string.IsNullOrWhiteSpace(task.AssigneeCode))
                await processTaskService.RefreshUserTaskCacheAsync(task.AssigneeCode, ct);

            // Если это часть группы — закрыть «родителя», когда все ознакомились
            if (task.ParentTaskId.HasValue)
            {
                var left = await unitOfWork.ProcessTaskRepository.CountAsync(
                    ct, t => t.ParentTaskId == task.ParentTaskId && t.Status == "Pending");

                if (left == 0)
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
using BpmBaseApi.Shared.Responses.Process;
using BpmBaseApi.Shared.Dtos;
using MediatR;
using Microsoft.AspNetCore.Mvc;
using Swashbuckle.AspNetCore.Annotations;

namespace BpmBaseApi.WebApi.Controllers
{
    [ApiController]
    [Route("api/process")]
    public class ProcessController : ControllerBase
    {
        private readonly IMediator mediator;
        public ProcessController(IMediator mediator) => this.mediator = mediator;

        /// <summary>Отправить на ознакомление</summary>
        [HttpPost("acquaintance/send")]
        [SwaggerResponse(StatusCodes.Status200OK, "Ок", typeof(BaseResponseDto<SendProcessResponse>))]
        public async Task<IActionResult> SendAcquaintance(
            [FromBody] SendAcquaintanceCommand command,
            CancellationToken cancellationToken)
        {
            var result = await mediator.Send(command, cancellationToken);
            return Ok(result);
        }

        /// <summary>Подтвердить «Ознакомлен»</summary>
        [HttpPost("acquaintance/ack")]
        [SwaggerResponse(StatusCodes.Status200OK, "Ок", typeof(BaseResponseDto<SendProcessResponse>))]
        public async Task<IActionResult> AcknowledgeAcquaintance(
            [FromBody] AcknowledgeAcquaintanceCommand command,
            CancellationToken cancellationToken)
        {
            var result = await mediator.Send(command, cancellationToken);
            return Ok(result);
        }
    }
}
