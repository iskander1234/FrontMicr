// BpmBaseApi.Domain/Entities/Event/Process/ProcessFileDeletedEvent.cs
using BpmBaseApi.Domain.SeedWork; // где у тебя BaseEvent

namespace BpmBaseApi.Domain.Entities.Event.Process
{
    public class ProcessFileDeletedEvent : BaseEvent
    {
        // Обычно BaseEvent уже содержит EntityId, но если нет — добавь:
        // public Guid EntityId { get; set; }
        // Можно (не обязательно) добавить ProcessDataId, если удобно в репозитории
        public Guid? ProcessDataId { get; set; }
    }
}
