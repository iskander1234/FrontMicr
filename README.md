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

        // Если нужно имя процесса — подтянем (опционально)
        var proc = await unitOfWork.ProcessRepository.GetByFilterAsync(ct, p => p.ProcessCode == pd.ProcessCode);

        var (recipients, comment) = TryReadFromPayload(cmd.PayloadJson);
        if (recipients.Count == 0)
            throw new HandlerException("В payload.acquaintance.recipients не найден ни один получатель", ErrorCodesEnum.Business);

        // Идемпотентность: не дублировать Pending по тем же пользователям
        var existingPending = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
            ct,
            t => t.ProcessDataId == pd.Id
                 && t.BlockCode == AcquaintanceCode
                 && t.Status == "Pending"
                 && !string.IsNullOrEmpty(t.AssigneeCode));

        var alreadyCodes = existingPending.Select(t => t.AssigneeCode!).ToHashSet(StringComparer.OrdinalIgnoreCase);
        var toCreate = recipients.Where(r => !alreadyCodes.Contains(r.UserCode)).ToList();
        if (toCreate.Count == 0)
            return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };

        // Родитель (скрытый)
        var parentId = Guid.NewGuid();
        await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
        {
            EntityId      = parentId,
            ProcessDataId = pd.Id,
            ProcessCode   = pd.ProcessCode,                 // <<< ОБЯЗАТЕЛЬНО
            ProcessName   = proc?.ProcessName,              // (опционально)
            RegNumber     = pd.RegNumber,                   // (опционально)
            Title         = pd.Title,                       // (опционально)
            BlockCode     = AcquaintanceCode,
            BlockName     = "Ознакомление",
            AssigneeCode  = "system",
            Status        = "Hidden",
            Comment       = comment
        }, ct);

        // Дочерние Pending-задачи
        foreach (var r in toCreate)
        {
            await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
            {
                EntityId      = Guid.NewGuid(),
                ParentTaskId  = parentId,
                ProcessDataId = pd.Id,
                ProcessCode   = pd.ProcessCode,             // <<< ОБЯЗАТЕЛЬНО
                ProcessName   = proc?.ProcessName,          // (опционально)
                RegNumber     = pd.RegNumber,               // (опционально)
                Title         = pd.Title,                   // (опционально)
                BlockCode     = AcquaintanceCode,
                BlockName     = "Ознакомление",
                AssigneeCode  = r.UserCode,
                AssigneeName  = r.UserName,
                Status        = "Pending",
                Comment       = comment
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

    private static (List<UserInfo> Recipients, string? Comment) TryReadFromPayload(Dictionary<string, object>? payload)
    {
        var recipients = new List<UserInfo>();
        string? comment = null;

        if (payload is null || payload.Count == 0) return (recipients, null);

        try
        {
            var json = JsonSerializer.Serialize(payload);
            var root = JsonNode.Parse(json)!.AsObject();
            var acq  = root["acquaintance"] as JsonObject;
            if (acq is null) return (recipients, null);

            // поддержим и "comment", и "message"
            comment = acq["comment"]?.GetValue<string>()
                   ?? acq["message"]?.GetValue<string>();

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
        catch { /* игнорируем кривой json */ }

        // dedupe
        recipients = recipients
            .GroupBy(x => x.UserCode, StringComparer.OrdinalIgnoreCase)
            .Select(g => g.First())
            .ToList();

        return (recipients, comment);
    }
}
