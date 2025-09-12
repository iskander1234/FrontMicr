// Rework → Submit: если прислали новый payload — мержим и сохраняем ProcessData
if (Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var currentStage)
    && currentStage == ProcessStage.Rework
    && command.Action == ProcessAction.Submit)
{
    if (command.PayloadJson is null)
        throw new HandlerException("Не передан PayloadJson для обновления.", ErrorCodesEnum.Business);

    // 1) сериализуем входной словарь
    var incomingJson = JsonSerializer.Serialize(command.PayloadJson);

    // 2) аккуратно накладываем изменения на текущий payload (deep-merge)
    var mergedJson = MergePayloadPreserve(processData.PayloadJson, incomingJson);

    // 3) если реально есть изменения — поднимаем событие
    if (!string.Equals(mergedJson, processData.PayloadJson, StringComparison.Ordinal))
    {
        await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataEditedEvent
        {
            EntityId      = processData.Id,
            StatusCode    = processData.StatusCode,
            StatusName    = processData.StatusName,
            PayloadJson   = mergedJson,
            InitiatorCode = processData.InitiatorCode,
            InitiatorName = processData.InitiatorName,
            Title         = processData.Title
            // StartedAt НЕ трогаем здесь, чтобы не сбивать начальный старт
        }, cancellationToken);

        // локально держим актуальную версию, чтобы дальше логика читала обновлённый JSON
        processData.PayloadJson = mergedJson;
    }
}




var pdPayload = JsonSerializer.Deserialize<Dictionary<string, object>>(processData.PayloadJson);
bool isSequential = ReadIsSequentialFromProcessData(pdPayload)

var pdPayload = JsonSerializer.Deserialize<Dictionary<string, object>>(processData.PayloadJson);
bool isSequential = ReadIsSequentialFromProcessData(pdPayload);


using System.Text.Encodings.Web;
using System.Text.Json.Nodes;

private static string MergePayloadPreserve(string baseJson, string patchJson)
{
    var opts = new JsonSerializerOptions { Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping };

    var baseObj  = ParseOrEmpty(baseJson);
    var patchObj = ParseOrEmpty(patchJson);

    DeepMerge(baseObj, patchObj);
    return JsonSerializer.Serialize(baseObj, opts);

    static JsonObject ParseOrEmpty(string? json)
    {
        if (string.IsNullOrWhiteSpace(json)) return new JsonObject();
        try { return (JsonNode.Parse(json) as JsonObject) ?? new JsonObject(); }
        catch { return new JsonObject(); }
    }

    static void DeepMerge(JsonObject target, JsonObject patch)
    {
        foreach (var kv in patch)
        {
            if (kv.Value is null)
            {
                // null в patch — перезаписываем как null (или можно пропускать, если так задумано)
                target[kv.Key] = null;
                continue;
            }

            if (kv.Value is JsonObject patchObj)
            {
                if (target[kv.Key] is JsonObject targetObj)
                {
                    DeepMerge(targetObj, patchObj);
                }
                else
                {
                    target[kv.Key] = patchObj.DeepClone();
                }
            }
            else
            {
                // массивы и примитивы — заменяем целиком
                target[kv.Key] = kv.Value.DeepClone();
            }
        }
    }
}
