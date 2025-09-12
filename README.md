payloadJson = SetRegnumInPayload(payloadJson, draft.RegNumber, jsonOptions);

await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataEditedEvent
{
    EntityId = draft.Id,
    StatusCode = "Started",
    StatusName = "В работе",
    PayloadJson = payloadJson,
    InitiatorCode = regData?.UserCode,
    InitiatorName = regData?.UserName,
    Title = processDataDto?.DocumentTitle ?? draft.Title,
    StartedAt = draft.Started
                        
}, cancellationToken);


с 
payloadJson = SetRegnumInPayload(payloadJson, draft.RegNumber, jsonOptions);
// ★ всегда фиксируем новый момент старта в payload
payloadJson = SetStartDateInPayload(payloadJson, DateTime.Now, jsonOptions);

await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataEditedEvent
{
    EntityId = draft.Id,
    StatusCode = "Started",
    StatusName = "В работе",
    PayloadJson = payloadJson,
    InitiatorCode = regData?.UserCode,
    InitiatorName = regData?.UserName,
    Title = processDataDto?.DocumentTitle ?? draft.Title,
    StartedAt = DateTime.Now // ★ ВСЕГДА новая дата при переходе из черновика
}, cancellationToken);




өөө
// NEW: вписываем номер в payload.regData.regnum
payloadJson = SetRegnumInPayload(payloadJson, requestNumber, jsonOptions);

var createdEvt = new ProcessDataCreatedEvent
{
    ...
    StatusCode = "Started",
    StatusName = "В работе",
    PayloadJson = payloadJson,
    Title = processDataDto.DocumentTitle,
    StartedAt = DateTime.Now
};


с 
// вписываем номер + дату старта в payload.regData
payloadJson = SetRegnumInPayload(payloadJson, requestNumber, jsonOptions);
payloadJson = SetStartDateInPayload(payloadJson, DateTime.Now, jsonOptions); // ★

var createdEvt = new ProcessDataCreatedEvent
{
    ...
    StatusCode = "Started",
    StatusName = "В работе",
    PayloadJson = payloadJson,
    Title = processDataDto.DocumentTitle,
    StartedAt = DateTime.Now // уже было ок
};




SetStartDateInPayload
static string SetStartDateInPayload(string payload, DateTime dt, JsonSerializerOptions opts)
{
    var root = JsonNode.Parse(payload)!.AsObject();
    if (root["regData"] is not JsonObject rd)
    {
        rd = new JsonObject();
        root["regData"] = rd;
    }

    // ISO 8601; если хочешь 'Z' — используй dt.ToUniversalTime().ToString("o")
    rd["startdate"] = dt.ToString("o");
    return JsonSerializer.Serialize(root, opts);
}


ProcessDataEntity
public DateTime? Started { get; set; } // новое поле

public void Apply(ProcessDataCreatedEvent @event)
{
    ...
    Started = @event.StartedAt; // ★
}

public void Apply(ProcessDataEditedEvent @event)
{
    ...
    if (@event.StartedAt != default) // или HasValue, если nullable
        Started = @event.StartedAt;  // ★ обновляем на новый старт
}



public class ProcessDataCreatedEvent : BaseEvent
{
    ...
    public DateTime StartedAt { get; set; } // ★
}

public class ProcessDataEditedEvent : BaseEvent
{
    ...
    public DateTime StartedAt { get; set; } // ★ можно сделать nullable, если хочешь
}
