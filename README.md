new CamundaSubmitTaskRequest
                    {
                        TaskId = claimedTasksResponse.TaskId,
                        Variables = GetCamundaVariablesForStage(stage, command.Condition, processData.Id)
                    });


                    private Dictionary<string, object> GetCamundaVariablesForStage(ProcessStage stage, ProcessCondition condition, Guid processDataId)
        {
            
            return stage switch
            {
                ProcessStage.Approval => new() { { "agreement", true } },
                ProcessStage.Signing => new() { { "sign", true } },
                _ => new()
            };
        }

            public enum ProcessCondition
    {
        /// <summary>
        /// Одобрить
        /// </summary>
        accept,

        /// <summary>
        /// На доработку
        /// </summary>
        remake,

        /// <summary>
        /// Отклонить
        /// </summary>
        reject
    }


    using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.SeedWork;

namespace BpmBaseApi.Domain.Entities.Process
{
    public class ProcessTaskHistoryEntity : BaseJournaledEntity
    {
        public Guid Id { get; set; }
        public Guid ProcessDataId { get; set; }
        public ProcessDataEntity ProcessData { get; set; }
        public Guid TaskId { get; set; }            
        public Guid? ParentTaskId { get; set; }
        public string Action { get; set; }             
        public string? AssigneeCode { get; set; }
        public string? AssigneeName { get; set; }
        public string? BlockCode { get; set; }
        public string BlockName { get; set; }          
        public string? Description { get; set; }
        public DateTime Timestamp { get; set; }
        public string? Comment { get; set; }
        public string? PayloadJson { get; set; }
        public string ProcessCode { get; set; }
        public string ProcessName { get; set; }
        public string RegNumber { get; set; }
        public string? InitiatorCode { get; set; }
        public string? InitiatorName { get; set; }
        public string? Title { get; set; }
        public string? Condition { get; set; }


        #region Apply

        public void Apply(ProcessTaskHistoryCreatedEvent @event)
        {
            Id = Guid.NewGuid();
            @event.EntityId = Id;
            TaskId = @event.TaskId;
            ParentTaskId = @event.ParentTaskId;
            Action = @event.Action;
            InitiatorCode = @event.InitiatorCode;
            InitiatorName = @event.InitiatorName;
            AssigneeCode = @event.AssigneeCode;
            AssigneeName = @event.AssigneeName;
            BlockCode = @event.BlockCode;
            BlockName = @event.BlockName;
            Description = @event.Description;
            ProcessDataId = @event.ProcessDataId;
            Comment = @event.Comment;
            PayloadJson = @event.PayloadJson;
            Timestamp = @event.Timestamp;
            ProcessCode = @event.ProcessCode;
            ProcessName = @event.ProcessName;
            RegNumber = @event.RegNumber;
            Title = @event.Title;
            Condition = @event.Condition;
        }

        #endregion
    }
}
  И смотри если ну он почему должен заходит в ProcessTaskHistoryEntity потому что если 3 человек 2 изних соглосовали а 1 изних нет то тогда надо удет fasle
