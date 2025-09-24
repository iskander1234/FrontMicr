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
