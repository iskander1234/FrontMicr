// Domain/Entities/Event/Process/ProcessDataEditedEvent.cs
using BpmBaseApi.Domain.Entities.Event;
using BpmBaseApi.Domain.SeedWork;

namespace BpmBaseApi.Domain.Entities.Event.Process
{
    public class ProcessDataEditedEvent : BaseEntityEvent
    {
        public string? StatusCode    { get; set; }  // "Started"
        public string? StatusName    { get; set; }  // "В работе"
        public string? PayloadJson   { get; set; }
        public string? InitiatorCode { get; set; }
        public string? InitiatorName { get; set; }
        public string? Title         { get; set; }
        // RegNumber — нарочно отсутствует
    }
}

// Domain/Entities/Process/ProcessDataEntity.cs (или где у тебя Apply)
public partial class ProcessDataEntity
{
    public void Apply(ProcessDataEditedEvent @event)
    {
        if (!string.IsNullOrEmpty(@event.StatusCode))     StatusCode    = @event.StatusCode!;
        if (!string.IsNullOrEmpty(@event.StatusName))     StatusName    = @event.StatusName!;
        if (!string.IsNullOrEmpty(@event.PayloadJson))    PayloadJson   = @event.PayloadJson!;
        if (!string.IsNullOrWhiteSpace(@event.InitiatorCode))  InitiatorCode = @event.InitiatorCode!;
        if (!string.IsNullOrWhiteSpace(@event.InitiatorName))  InitiatorName = @event.InitiatorName!;
        if (!string.IsNullOrWhiteSpace(@event.Title))     Title         = @event.Title!;
        // RegNumber — не трогаем
    }
}




// ... внутри StartProcessCommandHandler.Handle, в ветке когда нашли draft по regNumberFromPayload

// было:
// draft.StatusCode = "Started"; ... await unitOfWork.ProcessDataRepository.UpdateAsync(draft, cancellationToken);

// стало:
await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataEditedEvent
{
    EntityId       = draft.Id,
    StatusCode     = "Started",
    StatusName     = "В работе",
    PayloadJson    = payloadJson,
    InitiatorCode  = regData?.UserCode,
    InitiatorName  = regData?.UserName,
    Title          = processDataDto?.DocumentTitle ?? draft.Title
}, cancellationToken);

// дальше без изменений — стартуем Camunda на draft.Id, добавляем files (как у тебя), пишем историю и возвращаем ответ

