public async Task<bool> CamundaSetProcessVariables(
    string processInstanceId,
    Dictionary<string, object> variables)
{
    try
    {
        // защитимся от null/вложенных структур
        var cleaned = variables
            .Where(kv => kv.Value is not null && kv.Value is not IDictionary<string, object>)
            .ToDictionary(kv => kv.Key, kv => kv.Value!);

        if (cleaned.Count == 0)
            return true; // класть нечего — считаем успехом

        await camundaProvider.SetProcessVariablesAsync(processInstanceId, cleaned);
        return true;
    }
    catch (HandlerException) { throw; }
    catch (Exception ex)
    {
        throw new HandlerException($"Некорректный ответ от Camunda: {ex.Message} {ex.InnerException?.Message};");
    }
}





// Собираем classification (Emergency|Standard|Normal)
var variables = BuildClassificationVariables(command.PayloadJson); // вернёт {} если не нашли

if (variables.Count > 0)
{
    // ключевой шаг: положить в process-scope, чтобы шлюз увидел ${classification}
    await camundaService.CamundaSetProcessVariables(processData.ProcessInstanceId, variables);
}

if (string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
{
    var claimed = await camundaService.CamundaClaimedTasks(
        processData.ProcessInstanceId, parentTask.BlockCode)
        ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);

    // дальше submit уже можно слать без переменных
    var submit = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
    {
        TaskId = claimed.TaskId,
        Variables = new Dictionary<string, object>() // пусто OK
    });

    if (!submit.Success)
        throw new HandlerException($"Ошибка при отправке Msg:{submit.Msg}", ErrorCodesEnum.Camunda);

    await processTaskService.FinalizeTaskAsync(parentTask, ct);
}
