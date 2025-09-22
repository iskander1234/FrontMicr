// вместо CamundaVars(...) просто плоский словарь:
var vars = new Dictionary<string, object?>
{
    ["processGuid"]   = created.EntityId.ToString(),
    ["initiatorCode"] = command.InitiatorCode,
    ["initiatorName"] = command.InitiatorName,
    // English only, как и хотели: Emergency | Standard | Normal
    ["itsmLevel"]     = ReadItsmLevel(payloadJson) // null — просто не добавлять
}.Where(kv => kv.Value is not null)
 .ToDictionary(kv => kv.Key, kv => kv.Value!);

var pi = await camundaService.CamundaStartProcess(new CamundaStartProcessRequest
{
    processCode = command.ProcessCode,
    // businessKey можно добавить, если поддерживается интерфейсом
    variables   = vars
});
