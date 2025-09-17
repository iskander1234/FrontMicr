using System.Text.Json;
using System.Text.Json.Nodes;
using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Application.CommandHandlers.Process;

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

        var proc = await unitOfWork.ProcessRepository.GetByFilterAsync(ct, p => p.ProcessCode == pd.ProcessCode);

        // читаем получателей из payloadJson
        var (recipients, comment) = TryReadFromPayload(cmd.PayloadJson);
        if (recipients.Count == 0)
            throw new HandlerException("В payload.acquaintance.recipients нет получателей", ErrorCodesEnum.Business);

        // не дублируем уже висящие уведомления
        var existingNotify = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
            ct, t => t.ProcessDataId == pd.Id &&
                     t.BlockCode == AcquaintanceCode &&
                     (t.Status == "Notify" || t.Status == "Pending") &&
                     !string.IsNullOrEmpty(t.AssigneeCode));

        var alreadyCodes = existingNotify.Select(x => x.AssigneeCode!).ToHashSet(StringComparer.OrdinalIgnoreCase);
        var toCreate = recipients.Where(r => !alreadyCodes.Contains(r.UserCode)).ToList();
        if (toCreate.Count == 0)
            return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };

        // Родитель (скрытый контейнер) — без "system": ставим инициатора
        var parentId = Guid.NewGuid();
        await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
        {
            EntityId      = parentId,
            ProcessDataId = pd.Id,
            ProcessCode   = pd.ProcessCode,
            ProcessName   = proc?.ProcessName,
            RegNumber     = pd.RegNumber,
            Title         = pd.Title,
            BlockCode     = AcquaintanceCode,
            BlockName     = "Ознакомление",
            Status        = "Hidden",
            Comment       = comment,

            AssigneeCode  = pd.InitiatorCode,
            AssigneeName  = pd.InitiatorName,
            InitiatorCode = pd.InitiatorCode,
            InitiatorName = pd.InitiatorName
        }, ct, autoCommit: true); // фиксируем родителя, чтобы не ловить FK по ParentTaskId

        var parent = await unitOfWork.ProcessTaskRepository.GetByIdAsync(ct, parentId)
                    ?? throw new HandlerException("Не удалось создать родителя ознакомления", ErrorCodesEnum.Business);

        // Дочерние задания-уведомления (Notify)
        foreach (var r in toCreate)
        {
            var assigneeName = string.IsNullOrWhiteSpace(r.UserName) ? r.UserCode : r.UserName;

            await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
            {
                EntityId      = Guid.NewGuid(),
                ParentTaskId  = parent.Id,
                ProcessDataId = pd.Id,
                ProcessCode   = pd.ProcessCode,
                ProcessName   = proc?.ProcessName,
                RegNumber     = pd.RegNumber,
                Title         = pd.Title,
                BlockCode     = AcquaintanceCode,
                BlockName     = "Ознакомление",
                Status        = "Notify",           // <<< новый статус
                Comment       = comment,

                AssigneeCode  = r.UserCode,         // из payload
                AssigneeName  = assigneeName,       // из payload

                InitiatorCode = pd.InitiatorCode,
                InitiatorName = pd.InitiatorName
            }, ct);

            await processTaskService.RefreshUserTaskCacheAsync(r.UserCode, ct);
        }

        // История
        await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(new ProcessTaskHistoryCreatedEvent
        {
            ProcessDataId = pd.Id,
            TaskId        = parent.Id,
            BlockCode     = AcquaintanceCode,
            BlockName     = "Ознакомление",
            Action        = "SendForAcquaintance",
            ActionName    = "Отправлено на ознакомление",
            Timestamp     = DateTime.Now,
            Description   = string.Join(", ", toCreate.Select(x => $"{x.UserName}({x.UserCode})")),
            InitiatorCode = pd.InitiatorCode,
            InitiatorName = pd.InitiatorName
        }, ct);

        return new BaseResponseDto<SendProcessResponse>
        {
            Data = new SendProcessResponse { Success = true }
        };
    }

    private static (List<UserInfo> Recipients, string? Comment) TryReadFromPayload(Dictionary<string, object>? payload)
    {
        var recipients = new List<UserInfo>();
        string? comment = null;
        if (payload == null || payload.Count == 0) return (recipients, null);

        try
        {
            var json = JsonSerializer.Serialize(payload);
            var root = JsonNode.Parse(json)!.AsObject();
            var acq  = root["acquaintance"] as JsonObject;
            if (acq is null) return (recipients, null);

            comment = acq["comment"]?.GetValue<string>() ?? acq["message"]?.GetValue<string>();

            if (acq["recipients"] is JsonArray arr)
            {
                foreach (var n in arr.OfType<JsonObject>())
                {
                    var code = n["userCode"]?.GetValue<string>()?.Trim();
                    if (string.IsNullOrWhiteSpace(code)) continue;
                    var name = n["userName"]?.GetValue<string>();
                    recipients.Add(new UserInfo { UserCode = code!, UserName = name });
                }
            }
        }
        catch { /* ignore */ }

        return (
            recipients.GroupBy(x => x.UserCode, StringComparer.OrdinalIgnoreCase).Select(g => g.First()).ToList(),
            comment
        );
    }
}




using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Application.CommandHandlers.Process;

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
            throw new HandlerException("Эта задача не является задачей ознакомления", ErrorCodesEnum.Business);

        // идемпотентность: если уже не активная (ни Notify, ни Pending) — ок
        if (!(task.Status == "Notify" || task.Status == "Pending"))
            return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };

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

        await processTaskService.FinalizeTaskAsync(task, ct);
        if (!string.IsNullOrWhiteSpace(task.AssigneeCode))
            await processTaskService.RefreshUserTaskCacheAsync(task.AssigneeCode, ct);

        // закрыть родителя, если не осталось активных (Notify/Pending) детей
        if (task.ParentTaskId.HasValue)
        {
            var stillActive = await unitOfWork.ProcessTaskRepository.CountAsync(
                ct, t => t.ParentTaskId == task.ParentTaskId &&
                         (t.Status == "Notify" || t.Status == "Pending"));

            if (stillActive == 0)
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
