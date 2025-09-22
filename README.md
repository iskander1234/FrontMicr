public class SendProcessITSMCommandHandler(
    IUnitOfWork unitOfWork,
    ICamundaService camundaService,
    IProcessTaskService processTaskService)
    : IRequestHandler<SendProcessCommand, BaseResponseDto<SendProcessResponse>>
{
    public async Task<BaseResponseDto<SendProcessResponse>> Handle(
        SendProcessCommand command, CancellationToken ct)
    {
        var currentTask = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                              ct, t => t.Id == command.TaskId && t.Status == "Pending")
                          ?? throw new HandlerException("Задача не найдена или уже обработана", ErrorCodesEnum.Business);

        var processData = await unitOfWork.ProcessDataRepository.GetByIdAsync(ct, currentTask.ProcessDataId)
                         ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

        var parentTask = await unitOfWork.ProcessTaskRepository.GetByIdAsync(ct, currentTask.ParentTaskId!.Value);

        // Собираем classification (Emergency|Standard|Normal) из payloadJson
        var variables = BuildClassificationVariables(command.PayloadJson); // вернёт {} если не нашли

        if (string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
        {
            // Шлём переменные ВМЕСТЕ с submit — шлюзы увидят classification
            var claimed = await camundaService.CamundaClaimedTasks(
                              processData.ProcessInstanceId, parentTask.BlockCode)
                          ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);

            var submit = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
            {
                TaskId = claimed.TaskId,
                Variables = variables  // <= ВАЖНО
            });

            if (!submit.Success)
                throw new HandlerException($"Ошибка при отправке Msg:{submit.Msg}", ErrorCodesEnum.Camunda);

            await processTaskService.FinalizeTaskAsync(parentTask, ct);
        }
        else
        {
            // Родитель не system: просто поднимаем его
            await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
            {
                EntityId = parentTask.Id,
                Status = "Pending"
            }, ct);

            if (!string.IsNullOrWhiteSpace(parentTask.AssigneeCode))
                await processTaskService.RefreshUserTaskCacheAsync(parentTask.AssigneeCode, ct);
        }

        await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, ct);
        await processTaskService.FinalizeTaskAsync(currentTask, ct);
        await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, ct);

        return new BaseResponseDto<SendProcessResponse>
        {
            Data = new SendProcessResponse { Success = true }
        };
    }

    private static Dictionary<string, object> BuildClassificationVariables(Dictionary<string, object>? payload)
    {
        if (payload is null) return new();

        static string? ReadStr(Dictionary<string, object> p, string key)
        {
            if (!p.TryGetValue(key, out var v) || v is null) return null;
            var json = JsonSerializer.Serialize(v);
            return JsonSerializer.Deserialize<string>(json, new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
        }

        // Ищем processData.analyst.classificationCode
        string? fromAnalyst = null;
        if (payload.TryGetValue("processData", out var pdRaw) && pdRaw is not null)
        {
            var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(
                JsonSerializer.Serialize(pdRaw), new JsonSerializerOptions { PropertyNameCaseInsensitive = true });

            if (pd != null && pd.TryGetValue("analyst", out var aRaw) && aRaw is not null)
            {
                var analyst = JsonSerializer.Deserialize<Dictionary<string, object>>(
                    JsonSerializer.Serialize(aRaw), new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
                if (analyst != null) fromAnalyst = ReadStr(analyst, "classificationCode");
            }
        }

        var level = fromAnalyst
                    ?? ReadStr(payload, "classification")
                    ?? ReadStr(payload, "itsmLevel")
                    ?? ReadStr(payload, "level")
                    ?? ReadStr(payload, "priority");

        if (string.IsNullOrWhiteSpace(level)) return new();

        var canonical = level.Trim().ToLowerInvariant() switch
        {
            "emergency" => "Emergency",
            "standard"  => "Standard",
            "normal"    => "Normal",
            _ => null
        };

        return canonical is null ? new() : new() { ["classification"] = canonical };
    }
}
