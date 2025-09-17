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

        // 1) получатели из payloadJson
        var (recipients, comment) = TryReadFromPayload(cmd.PayloadJson);
        if (recipients.Count == 0)
            throw new HandlerException("В payload.acquaintance.recipients нет получателей", ErrorCodesEnum.Business);

        // 2) не дублируем активные (Notify|Pending)
        var existingActive = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
            ct, t => t.ProcessDataId == pd.Id &&
                     t.BlockCode == AcquaintanceCode &&
                     (t.Status == "Notify" || t.Status == "Pending") &&
                     !string.IsNullOrEmpty(t.AssigneeCode));

        var already = existingActive.Select(x => x.AssigneeCode!).ToHashSet(StringComparer.OrdinalIgnoreCase);
        var toCreate = recipients.Where(r => !already.Contains(r.UserCode)).ToList();
        if (toCreate.Count == 0)
            return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };

        // 3) создаём задачи-уведомления (только «дети», без родителя)
        foreach (var r in toCreate)
        {
            var assigneeName = string.IsNullOrWhiteSpace(r.UserName) ? r.UserCode : r.UserName;

            await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
            {
                EntityId      = Guid.NewGuid(),
                ProcessDataId = pd.Id,

                // важно для not-null
                ProcessCode   = pd.ProcessCode,
                ProcessName   = proc?.ProcessName,
                RegNumber     = pd.RegNumber,
                Title         = pd.Title,

                BlockCode     = AcquaintanceCode,
                BlockName     = "Ознакомление",
                Status        = "Notify", // обрабатывай как Pending в инбоксе
                Comment       = comment,

                AssigneeCode  = r.UserCode,
                AssigneeName  = assigneeName,

                InitiatorCode = pd.InitiatorCode,
                InitiatorName = pd.InitiatorName
            }, ct);

            await processTaskService.RefreshUserTaskCacheAsync(r.UserCode, ct);
        }

        // 4) запись в историю (общая)
        await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(new ProcessTaskHistoryCreatedEvent
        {
            ProcessDataId = pd.Id,
            TaskId        = pd.Id, // без родителя можно сослаться на PD
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

        recipients = recipients
            .GroupBy(x => x.UserCode, StringComparer.OrdinalIgnoreCase)
            .Select(g => g.First())
            .ToList();

        return (recipients, comment);
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
    private static bool IsActive(string? s) => s == "Notify" || s == "Pending";

    public async Task<BaseResponseDto<SendProcessResponse>> Handle(AcknowledgeAcquaintanceCommand cmd, CancellationToken ct)
    {
        var task = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(ct, t => t.Id == cmd.TaskId)
                   ?? throw new HandlerException("Задача не найдена", ErrorCodesEnum.Business);

        if (!string.Equals(task.BlockCode, AcquaintanceCode, StringComparison.OrdinalIgnoreCase))
            throw new HandlerException("Эта задача не является задачей ознакомления", ErrorCodesEnum.Business);

        if (!IsActive(task.Status))
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

        return new BaseResponseDto<SendProcessResponse>
        {
            Data = new SendProcessResponse { Success = true }
        };
    }
}
