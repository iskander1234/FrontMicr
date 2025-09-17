using System.Text.Json;
using System.Text.Json.Nodes;
using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Enum;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Application.CommandHandlers.Process;

// SEND
public class SendAcquaintanceCommandHandler(
    IUnitOfWork unitOfWork,
    IProcessTaskService processTaskService
) : IRequestHandler<SendAcquaintanceCommand, BaseResponseDto<SendProcessResponse>>
{
    private static readonly string BlockCode = ProcessStage.Acquaintance.ToString();

    public async Task<BaseResponseDto<SendProcessResponse>> Handle(SendAcquaintanceCommand cmd, CancellationToken ct)
    {
        var pd = await unitOfWork.ProcessDataRepository.GetByIdAsync(ct, cmd.ProcessDataId)
                 ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

        // (опционально) подтянуть имя процесса
        var proc = await unitOfWork.ProcessRepository.GetByFilterAsync(ct, p => p.ProcessCode == pd.ProcessCode);

        var (recipients, comment) = TryReadFromPayload(cmd.PayloadJson);
        if (recipients.Count == 0)
            throw new HandlerException("В payload.acquaintance.recipients не найден ни один получатель",
                ErrorCodesEnum.Business);

        // защитимся от дублей: не создавать повторные Pending для тех же пользователей
        var existingPending = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
            ct,
            t => t.ProcessDataId == pd.Id
                 && t.BlockCode == BlockCode
                 && t.Status == "Pending"
                 && !string.IsNullOrEmpty(t.AssigneeCode));

        var already = existingPending.Select(t => t.AssigneeCode!).ToHashSet(StringComparer.OrdinalIgnoreCase);
        var toCreate = recipients.Where(r => !already.Contains(r.UserCode)).ToList();
        if (toCreate.Count == 0)
            return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };

        // родитель (скрытый)
        var parentId = Guid.NewGuid();
        await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
        {
            EntityId      = parentId,
            ProcessDataId = pd.Id,
            ProcessCode   = pd.ProcessCode,          // NOT NULL в БД
            ProcessName   = proc?.ProcessName,
            RegNumber     = pd.RegNumber,
            Title         = pd.Title,
            BlockCode     = BlockCode,
            BlockName     = "Ознакомление",
            AssigneeCode  = "system",
            AssigneeName  = "system",                // NOT NULL в БД
            Status        = "Hidden",
            Comment       = comment
        }, ct);

        // дочерние Pending-задачи
        foreach (var r in toCreate)
        {
            var code = r.UserCode.Trim();
            var name = string.IsNullOrWhiteSpace(r.UserName) ? code : r.UserName!.Trim(); // фолбэк

            await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
            {
                EntityId      = Guid.NewGuid(),
                ParentTaskId  = parentId,
                ProcessDataId = pd.Id,
                ProcessCode   = pd.ProcessCode,      // NOT NULL
                ProcessName   = proc?.ProcessName,
                RegNumber     = pd.RegNumber,
                Title         = pd.Title,
                BlockCode     = BlockCode,
                BlockName     = "Ознакомление",
                AssigneeCode  = code,
                AssigneeName  = name,                // NOT NULL
                Status        = "Pending",
                Comment       = comment
            }, ct);

            await processTaskService.RefreshUserTaskCacheAsync(code, ct);
        }

        // история (action — строка, не enum)
        await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(new ProcessTaskHistoryCreatedEvent
        {
            ProcessDataId = pd.Id,
            TaskId        = parentId,
            BlockCode     = BlockCode,
            BlockName     = "Ознакомление",
            Action        = "SendForAcquaintance",
            ActionName    = "Отправлено на ознакомление",
            Timestamp     = DateTime.Now,
            Description   = string.Join(", ", toCreate.Select(x => $"{x.UserName ?? x.UserCode}({x.UserCode})"))
        }, ct);

        return new BaseResponseDto<SendProcessResponse>
        {
            Data = new SendProcessResponse { Success = true }
        };
    }

    private static (List<UserInfo> Recipients, string? Comment) TryReadFromPayload(Dictionary<string, object>? payload)
    {
        var list = new List<UserInfo>();
        string? comment = null;

        if (payload is null || payload.Count == 0) return (list, null);

        try
        {
            var json = JsonSerializer.Serialize(payload);
            var root = JsonNode.Parse(json)!.AsObject();
            var acq  = root["acquaintance"] as JsonObject;
            if (acq is null) return (list, null);

            comment = acq["comment"]?.GetValue<string>() ?? acq["message"]?.GetValue<string>();

            if (acq["recipients"] is JsonArray arr)
            {
                foreach (var n in arr.OfType<JsonObject>())
                {
                    var code = n["userCode"]?.GetValue<string>()?.Trim();
                    if (string.IsNullOrWhiteSpace(code)) continue;
                    var name = n["userName"]?.GetValue<string>();
                    list.Add(new UserInfo { UserCode = code!, UserName = name });
                }
            }
        }
        catch { /* игнорируем кривой JSON */ }

        // dedupe
        list = list.GroupBy(x => x.UserCode, StringComparer.OrdinalIgnoreCase).Select(g => g.First()).ToList();
        return (list, comment);
    }
}
using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Application.CommandHandlers.Process;

// ACK
public class AcknowledgeAcquaintanceCommandHandler(
    IUnitOfWork unitOfWork,
    IProcessTaskService processTaskService
) : IRequestHandler<AcknowledgeAcquaintanceCommand, BaseResponseDto<SendProcessResponse>>
{
    private static readonly string BlockCode = ProcessStage.Acquaintance.ToString();

    public async Task<BaseResponseDto<SendProcessResponse>> Handle(AcknowledgeAcquaintanceCommand cmd, CancellationToken ct)
    {
        var task = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(ct, t => t.Id == cmd.TaskId)
                   ?? throw new HandlerException("Задача не найдена", ErrorCodesEnum.Business);

        if (!string.Equals(task.BlockCode, BlockCode, StringComparison.OrdinalIgnoreCase))
            throw new HandlerException("Эта задача не является задачей ознакомления", ErrorCodesEnum.Business);

        // идемпотентность
        if (!string.Equals(task.Status, "Pending", StringComparison.OrdinalIgnoreCase))
            return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };

        // история
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

        // финализация
        await processTaskService.FinalizeTaskAsync(task, ct);
        if (!string.IsNullOrWhiteSpace(task.AssigneeCode))
            await processTaskService.RefreshUserTaskCacheAsync(task.AssigneeCode, ct);

        // закрыть родителя, если все ознакомились
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
