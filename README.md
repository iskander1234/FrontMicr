var updatedPayload = UpsertClassificationIntoJson(processData.PayloadJson, incomingClassification!);

// если реально изменилось — поднимем ProcessDataEditedEvent
if (!string.Equals(updatedPayload, processData.PayloadJson, StringComparison.Ordinal))
{
    await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataEditedEvent
    {
        EntityId      = processData.Id,
        StatusCode    = processData.StatusCode,
        StatusName    = processData.StatusName,
        PayloadJson   = updatedPayload,
        InitiatorCode = processData.InitiatorCode,
        InitiatorName = processData.InitiatorName,
        Title         = processData.Title   // заголовок не меняем (или посчитай, если нужно)
        // StartedAt, ProcessCode и пр. — как у тебя обрабатывается в аплаере
    }, ct);

    // держим актуальную версию локально
    processData.PayloadJson = updatedPayload;
}







private static string UpsertClassificationIntoJson(string? json, string value)
{
    var root = (JsonNode.Parse(string.IsNullOrWhiteSpace(json) ? "{}" : json) as JsonObject) ?? new JsonObject();

    var pd = root.EnsureObject("processData");
    var analyst = pd.EnsureObject("analyst");
    analyst["classificationCode"] = value;

    if (root.ContainsKeyIgnoreCase("classification"))
        root.SetStringCaseInsensitive("classification", value);
    if (pd.ContainsKeyIgnoreCase("classification"))
        pd.SetStringCaseInsensitive("classification", value);

    return root.ToJsonString();
}
