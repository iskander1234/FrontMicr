public interface ICamundaService
{
    Task<bool> CamundaSetProcessVariables(string processInstanceId, Dictionary<string, object> variables);
}


if (variables.Count > 0)
{
    var ok = await camundaService.CamundaSetProcessVariables(processData.ProcessInstanceId, variables);
    if (!ok) throw new HandlerException("Не удалось установить переменные процесса", ErrorCodesEnum.Camunda);
}
