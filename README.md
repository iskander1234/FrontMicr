using System.Diagnostics;
using System.Reflection.Emit;
using BpmBaseApi.Domain.Entities.Event.Process;
using System.Security.Cryptography;
using BpmBaseApi.Domain.SeedWork;

namespace BpmBaseApi.Domain.Entities.Process
{
    public class ProcessTaskEntity : BaseJournaledEntity
    {
        public Guid Id { get; set; }
        public Guid ProcessDataId { get; set; }
        public ProcessDataEntity ProcessData { get; set; }
        public Guid? ParentTaskId { get; set; }
        public ProcessTaskEntity? ParentTask { get; set; }
        public List<ProcessTaskEntity> SubTasks { get; set; } = new ();
        public Guid BlockId { get; set; }
        public BlockEntity Block { get; set; }
        public string? BlockCode { get; set; }
        public string BlockName { get; set; }
        public string AssigneeCode { get; set; }
        public string AssigneeName { get; set; }
        public string Status { get; set; }    
        public string? Comment { get; set; }
        public string? PayloadJson { get; set; }
        public string ProcessCode { get; set; }
        public string ProcessName { get; set; }
        public string RegNumber { get; set; }
        public string InitiatorCode { get; set; }
        public string InitiatorName { get; set; }
        public string Title { get; set; }


        #region Apply

        public void Apply(ProcessTaskCreatedEvent @event)
        {
            Id = Guid.NewGuid();
            @event.EntityId = Id;
            ParentTaskId = @event.ParentTaskId;
            ProcessDataId = @event.ProcessDataId;
            BlockId = @event.BlockId;
            BlockCode = @event.BlockCode;
            BlockName = @event.BlockName;
            AssigneeCode = @event.AssigneeCode;
            AssigneeName = @event.AssigneeName;
            Status = @event.Status;
            Comment = @event.Comment;
            PayloadJson = @event.PayloadJson;
            ProcessCode = @event.ProcessCode;
            ProcessName = @event.ProcessName;
            RegNumber = @event.RegNumber;
            InitiatorCode = @event.InitiatorCode;
            InitiatorName = @event.InitiatorName;
            Title = @event.Title;
        }

        public void Apply(ProcessTaskStatusChangedEvent @event)
        {
            Status = @event.Status;
        }

        #endregion
    }
}
