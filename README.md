// стало (чуть безопаснее с TryAdd):
var vars = new Dictionary<string, object?>(StringComparer.Ordinal)
{
    ["processGuid"]   = created.EntityId.ToString(),
    ["initiatorCode"] = string.IsNullOrWhiteSpace(command.InitiatorCode) ? null : command.InitiatorCode,
    ["initiatorName"] = string.IsNullOrWhiteSpace(command.InitiatorName) ? null : command.InitiatorName
};
var level = ReadItsmLevel(payloadJson);
if (!string.IsNullOrWhiteSpace(level)) vars["itsmLevel"] = level;

var cleaned = vars.Where(kv => kv.Value is not null)
                  .ToDictionary(kv => kv.Key, kv => kv.Value!);

var pi = await camundaService.CamundaStartProcess(new CamundaStartProcessRequest
{
    processCode = command.ProcessCode,
    variables   = cleaned
    // businessKey = created.EntityId.ToString(), // включить, если поддерживается
});


if (cleaned.Values.Any(v => v is IDictionary<string, object>))
    throw new HandlerException("Camunda variables must be plain key/value.", ErrorCodesEnum.Business);
