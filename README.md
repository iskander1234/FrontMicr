// Отправить на ознакомление
public record SendAcquaintanceCommand(
    Guid ProcessDataId,
    List<UserInfo> Recipients,     // UserCode/UserName
    string? Message                // опционально
) : IRequest<BaseResponseDto<SendProcessResponse>>;

// Пометить “Ознакомлен”
public record AcknowledgeAcquaintanceCommand(
    Guid TaskId                    // Id подзадачи ознакомления
) : IRequest<BaseResponseDto<SendProcessResponse>>;




using MediatR;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Services.Interfaces;

public class SendAcquaintanceCommandHandler(
    IUnitOfWork unitOfWork,
    IProcessTaskService processTaskService
) : IRequestHandler<SendAcquaintanceCommand, BaseResponseDto<SendProcessResponse>>
{
    public async Task<BaseResponseDto<SendProcessResponse>> Handle(SendAcquaintanceCommand cmd, CancellationToken ct)
    {
        var pd = await unitOfWork.ProcessDataRepository.GetByIdAsync(ct, cmd.ProcessDataId)
                 ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

        // (не обязательно) создаём «родителя» группы, чтобы при желании закрывать группу целиком
        var parentId = Guid.NewGuid();
        await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
        {
            EntityId     = parentId,
            ProcessDataId= pd.Id,
            BlockCode    = "Acquaintance",
            BlockName    = "Ознакомление",
            AssigneeCode = "system",
            Status       = "Hidden",         // не попадает в инбокс пользователей
            Comment      = cmd.Message
        }, ct);

        // dedupe получателей по UserCode
        var recipients = (cmd.Recipients ?? new()).Where(r => !string.IsNullOrWhiteSpace(r.UserCode))
                                                  .GroupBy(r => r.UserCode)
                                                  .Select(g => g.First())
                                                  .ToList();

        foreach (var r in recipients)
        {
            await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
            {
                EntityId      = Guid.NewGuid(),
                ParentTaskId  = parentId,
                ProcessDataId = pd.Id,
                BlockCode     = "Acquaintance",
                BlockName     = "Ознакомление",
                AssigneeCode  = r.UserCode,
                AssigneeName  = r.UserName,
                Status        = "Pending",
                Comment       = cmd.Message
            }, ct);

            // чтобы у адресата мгновенно появилась карточка
            await processTaskService.RefreshUserTaskCacheAsync(r.UserCode, ct);
        }

        // (опционально) записать историю "Отправлено на ознакомление"
        await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(new ProcessTaskHistoryCreatedEvent
        {
            ProcessDataId = pd.Id,
            TaskId        = parentId,
            BlockCode     = "Acquaintance",
            BlockName     = "Ознакомление",
            Action        = "SendForAcquaintance",
            ActionName    = "Отправлено на ознакомление",
            Timestamp     = DateTime.Now,
            Description   = string.Join(", ", recipients.Select(x => $"{x.UserName}({x.UserCode})"))
        }, ct);

        return new BaseResponseDto<SendProcessResponse>
        {
            Data = new SendProcessResponse { Success = true }
        };
    }
}



using MediatR;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Services.Interfaces;

public class AcknowledgeAcquaintanceCommandHandler(
    IUnitOfWork unitOfWork,
    IProcessTaskService processTaskService
) : IRequestHandler<AcknowledgeAcquaintanceCommand, BaseResponseDto<SendProcessResponse>>
{
    public async Task<BaseResponseDto<SendProcessResponse>> Handle(AcknowledgeAcquaintanceCommand cmd, CancellationToken ct)
    {
        var task = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(ct, t => t.Id == cmd.TaskId)
                   ?? throw new HandlerException("Задача не найдена", ErrorCodesEnum.Business);

        if (!string.Equals(task.BlockCode, "Acquaintance", StringComparison.OrdinalIgnoreCase))
            throw new HandlerException("Эта задача не является задачей ознакомления", ErrorCodesEnum.Business);

        // допускаем идемпотентность: если уже финализировали — возвращаем успех
        if (!string.Equals(task.Status, "Pending", StringComparison.OrdinalIgnoreCase))
            return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };

        // лог истории (как в других местах)
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

        // финализируем задачу (убираем из инбокса)
        await processTaskService.FinalizeTaskAsync(task, ct);
        if (!string.IsNullOrWhiteSpace(task.AssigneeCode))
            await processTaskService.RefreshUserTaskCacheAsync(task.AssigneeCode, ct);

        // если была группа — закрыть «родителя», когда все ознакомились
        if (task.ParentTaskId.HasValue)
        {
            var pendingLeft = await unitOfWork.ProcessTaskRepository.CountAsync(
                ct, t => t.ParentTaskId == task.ParentTaskId && t.Status == "Pending");

            if (pendingLeft == 0)
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
