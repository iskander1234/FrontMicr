StartProcessCommandHandler
____

var payloadJson = JsonSerializer.Serialize(command.Payload, options);
_


// NEW: гарантируем наличие sysInfo.sequence (false по умолчанию)
var rootObj = JsonNode.Parse(payloadJson)!.AsObject();
var sysInfo = rootObj["sysInfo"] as JsonObject;
if (sysInfo == null)
{
    sysInfo = new JsonObject();
    rootObj["sysInfo"] = sysInfo;
}
if (!sysInfo.ContainsKey("sequence"))
{
    sysInfo["sequence"] = false; // по умолчанию параллельно
}
// перезапишем payloadJson, чтобы в БД легло уже с sequence
payloadJson = rootObj.ToJsonString(new JsonSerializerOptions { Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping });


SetProcessStageCommandHandler
_____
var systemTask = await processTaskService.CreateSystemTaskAsync(...);
await processTaskService.CreateTasksNewAsync(...); // как у тебя сейчас
_


// NEW: определяем последовательный режим по payload
bool sequential = IsSequential(payloadDict);

if (sequential)
{
    // Получаем всех детей данного systemTask
    var children = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
        cancellationToken,
        t => t.ParentTaskId == systemTask.EntityId
    );
    if (children?.Count > 0)
    {
        var ordered = children.OrderBy(t => t.Created).ToList();
        var first = ordered.First();

        // Первая — Pending
        await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
        {
            EntityId = first.Id,
            Status = "Pending"
        }, cancellationToken);

        // Остальные — Waiting
        foreach (var w in ordered.Skip(1))
        {
            await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
            {
                EntityId = w.Id,
                Status = "Waiting"
            }, cancellationToken);
        }
    }
}
// иначе (parallel) — ничего не делаем, старое поведение остаётся


private bool IsSequential(Dictionary<string, object>? payload)
{
    if (payload == null) return false;
    if (!payload.TryGetValue("sysInfo", out var sysRaw) || sysRaw is null) return false;

    var json = JsonSerializer.Serialize(sysRaw);
    var sys = JsonSerializer.Deserialize<Dictionary<string, object>>(json);

    if (sys != null && sys.TryGetValue("sequence", out var v) && v is not null)
    {
        if (v is bool b) return b;
        if (bool.TryParse(v.ToString(), out var parsed)) return parsed;
    }
    return false;
}



SendProcessCommandHandler

Вставить (до pendingSiblings) 

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
             ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camунда);

        if (!Enum.TryParse<ProcessStage>(parentTaskSeq.BlockCode, out var stageSeq))
            throw new InvalidOperationException($"Unknown process stage: {parentTaskSeq.BlockCode}");

        var submit = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
        {
            TaskId = claimed.TaskId,
            Variables = GetCamundaVariablesForStage(stageSeq)
        });

        if (!submit.Success)
            throw new HandlerException($"Ошибка при отправке Msg:{submit.Msg}", ErrorCodesEnum.Camунда);

        await processTaskService.FinalizeTaskAsync(parentTaskSeq, cancellationToken);
    }

    return new BaseResponseDto<SendProcessResponse>
    {
        Data = new SendProcessResponse { Success = true }
    };
}
// === дальше остаётся твоя существующая параллельная логика ===



