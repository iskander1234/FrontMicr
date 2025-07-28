using BpmBaseApi.Domain.SeedWork;

namespace BpmBaseApi.Domain.Entities.Event.Process
{
    public class ProcessTaskCreatedEvent : BaseEntityEvent
    {
        public Guid? ParentTaskId { get; set; }
        public Guid BlockId { get; set; }
        public string BlockCode { get; set; }
        public string BlockName { get; set; }
        public string AssigneeCode { get; set; }
        public string AssigneeName { get; set; }
        public string Status { get; set; }
        public string? Comment { get; set; }
        public string? PayloadJson { get; set; }
        public Guid ProcessDataId { get; set; }
        public string ProcessCode { get; set; }
        public string ProcessName { get; set; }
        public string RegNumber { get; set; }
        public string InitiatorCode { get; set; }
        public string InitiatorName { get; set; }
        public string Title { get; set; }

    }
}
