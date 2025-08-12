using System.Text.Json;

namespace BpmBaseApi.Application.Helpers
{
    public static class TaskRoutingHelper
    {
        /// <summary>Читаем флажок последовательности из payload.sysInfo.sequence</summary>
        public static bool ResolveSequential(Dictionary<string, object> payload)
        {
            if (!payload.TryGetValue("sysInfo", out var sys) || sys is null) return false;
            var json = JsonSerializer.Serialize(sys);
            var map  = JsonSerializer.Deserialize<Dictionary<string, object>>(json) ?? new();
            return map.TryGetValue("sequence", out var v)
                && bool.TryParse(v?.ToString(), out var b)
                && b;
        }

        /// <summary>Вставляем в JSON задачи блок taskRouting {mode, seqIndex}</summary>
        public static string InjectTaskRoutingToPayload(string basePayloadJson, string mode, int seqIndex)
        {
            var dict = string.IsNullOrWhiteSpace(basePayloadJson)
                ? new Dictionary<string, object>()
                : (JsonSerializer.Deserialize<Dictionary<string, object>>(basePayloadJson) ?? new());

            dict["taskRouting"] = new Dictionary<string, object>
            {
                { "mode", mode },
                { "seqIndex", seqIndex }
            };

            return JsonSerializer.Serialize(dict);
        }

        /// <summary>Читаем из JSON задачи taskRouting {mode, seqIndex}</summary>
        public static (bool sequential, int? seqIndex) ReadTaskRouting(string? taskPayload)
        {
            if (string.IsNullOrWhiteSpace(taskPayload)) return (false, null);
            try
            {
                var dict = JsonSerializer.Deserialize<Dictionary<string, object>>(taskPayload!) ?? new();
                if (!dict.TryGetValue("taskRouting", out var tr) || tr is null) return (false, null);
                var s = JsonSerializer.Serialize(tr);
                var m = JsonSerializer.Deserialize<Dictionary<string, object>>(s) ?? new();
                var mode = m.TryGetValue("mode", out var md) ? md?.ToString() : null;

                int? idx = null;
                if (m.TryGetValue("seqIndex", out var si) && int.TryParse(si?.ToString(), out var parsed)) idx = parsed;

                return ("sequential".Equals(mode, StringComparison.OrdinalIgnoreCase), idx);
            }
            catch { return (false, null); }
        }
    }
}


_____________

var systemTask = await processTaskService.CreateSystemTaskAsync(processStageInfo.Code, processStageInfo.Name, processData, cancellationToken);

await processTaskService.CreateTasksNewAsync(processStageInfo.Code, processStageInfo.Name, recipients, processData, systemTask.EntityId, cancellationToken);

____________




using BpmBaseApi.Application.Helpers; // добавьте вверху файла

var systemTask = await processTaskService.CreateSystemTaskAsync(
    processStageInfo.Code, processStageInfo.Name, processData, cancellationToken);

// определяем последовательный/параллельный из processData.PayloadJson
var payloadDict = JsonSerializer.Deserialize<Dictionary<string, object>>(processData.PayloadJson);
var isSequential = TaskRoutingHelper.ResolveSequential(payloadDict ?? new());

// создаём задачи исполнителям по порядку массива recipients
int seq = 1;
foreach (var r in recipients)
{
    var assigneeCode = r.Login ?? r.UserCode; // что у вас стабильно заполнено
    var assigneeName = r.Name;

    // первая задача в последовательном режиме — Pending, остальные Waiting
    var status = (!isSequential || seq == 1) ? "Pending" : "Waiting";

    // вставляем служебный taskRouting в payload задачи (только для последовательного)
    var taskPayload = isSequential
        ? TaskRoutingHelper.InjectTaskRoutingToPayload(processData.PayloadJson, "sequential", seq)
        : processData.PayloadJson;

    await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
    {
        ProcessDataId = processData.Id,
        ParentTaskId  = systemTask.EntityId,
        BlockCode     = processStageInfo.Code,
        BlockName     = processStageInfo.Name,
        AssigneeCode  = assigneeCode,
        AssigneeName  = assigneeName,
        Status        = status,
        PayloadJson   = taskPayload,
        ProcessCode   = processData.ProcessCode,
        ProcessName   = processData.ProcessName,
        RegNumber     = processData.RegNumber,
        InitiatorCode = processData.InitiatorCode,
        InitiatorName = processData.InitiatorName,
        Title         = processData.Title
    }, cancellationToken);

    await processTaskService.RefreshUserTaskCacheAsync(assigneeCode, cancellationToken);

    if (isSequential) seq++;
}






// ===== Делегирование =====
if (command.Action == ProcessAction.Delegate && recipients.Any())
{
    await processTaskService.HandleDelegationAsync(currentTask, processData, command, recipients, comment, cancellationToken);
    return processTaskService.SuccessResponse(currentTask.BlockCode, currentTask.BlockName);
}



using BpmBaseApi.Application.Helpers;        // добавьте вверху файла
using BpmBaseApi.Domain.Entities.Process;    // если ещё не подключено
using System.Linq;                           // для OrderBy

// === NEW: если текущая задача в последовательной группе — передаём «маяк» следующему ===
if (currentTask.ParentTaskId != null)
{
    var (isSeq, currIdx) = TaskRoutingHelper.ReadTaskRouting(currentTask.PayloadJson);
    if (isSeq)
    {
        // 1) лог/закрытие текущей
        await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
        await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

        // 2) братья по той же группе
        var siblings = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
            cancellationToken, t => t.ParentTaskId == currentTask.ParentTaskId);

        // 3) найдём следующего с Waiting и seqIndex > текущего
        var next = siblings
            .Select(s => new { Task = s, Routing = TaskRoutingHelper.ReadTaskRouting(s.PayloadJson) })
            .Where(x => x.Routing.sequential
                        && x.Task.Status == "Waiting"
                        && x.Routing.seqIndex.HasValue
                        && (!currIdx.HasValue || x.Routing.seqIndex.Value > currIdx.Value))
            .OrderBy(x => x.Routing.seqIndex!.Value)
            .Select(x => x.Task)
            .FirstOrDefault();

        if (next != null)
        {
            await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
            {
                EntityId = next.Id,
                Status   = "Pending"
            }, cancellationToken);

            await processTaskService.RefreshUserTaskCacheAsync(next.AssigneeCode, cancellationToken);

            return new BaseResponseDto<SendProcessResponse>
            {
                Data = new SendProcessResponse { Success = true }
            };
        }

        // 4) если следующего нет — вся последовательная группа завершена
        // дальше течёт ваша старая логика (родитель system, Camunda и т.д.)
    }
}



"sysInfo": {
  "userCode": "…",
  "userName": "…",
  "comment": "…",
  "action": "submit",
  "condition": "string",
  "sequence": true  // ← включает последовательный режим
}


