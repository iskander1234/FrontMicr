namespace BpmBaseApi.Domain.Entities.Event.Process
{
    public class ProcessDataPayloadChangedEvent
    {
        public Guid EntityId { get; set; }        // Id ProcessData
        public string PayloadJson { get; set; } = "";
        public string? Title { get; set; }        // можно null — тогда не менять Title
    }
}
public void Apply(ProcessDataPayloadChangedEvent @event)
    {
        PayloadJson = @event.PayloadJson;
        if (!string.IsNullOrWhiteSpace(@event.Title))
            Title = @event.Title!;
    }
