using AutoMapper;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Queries.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Application.QueryHandlers.Process
{
    public class GetProcessHistoryQueryHandler(
    IUnitOfWork unitOfWork,
    IMapper mapper,
    IProcessService processService
) : IRequestHandler<GetProcessHistoryQuery, BaseResponseDto<List<GetProcessHistoryResponse>>>
    {
        public async Task<BaseResponseDto<List<GetProcessHistoryResponse>>> Handle(GetProcessHistoryQuery query, CancellationToken cancellationToken)
        {
            var processTask = await unitOfWork.ProcessTaskRepository
                .GetByFilterAsync(cancellationToken, h => h.Id == query.TaskId);

            var history = await unitOfWork.ProcessTaskHistoryRepository
                .GetByFilterListAsync(cancellationToken, h => h.ProcessDataId == processTask.ProcessDataId);

            var historyItems = mapper.Map<List<GetProcessHistoryResponse>>(history.OrderBy(h => h.Timestamp));

            var tree = processService.BuildHistoryTree(historyItems);

            return new BaseResponseDto<List<GetProcessHistoryResponse>> { Data = tree };
        }
    }
}

public class ProcessTaskEntity : BaseJournaledEntity
    {
        public Guid Id { get; set; }
        public Guid ProcessDataId { get; set; }
        public ProcessDataEntity ProcessData { get; set; }
        public Guid? ParentTaskId { get; set; }
        public ProcessTaskEntity? ParentTask { get; set; }
        public List<ProcessTaskEntity> SubTasks { get; set; } = new ();
        public Guid? BlockId { get; set; }
        //public BlockEntity Block { get; set; }
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
        public int? Order { get; set; }


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
            Order = @event.Order;
        }

        public void Apply(ProcessTaskStatusChangedEvent @event)
        {
            Status = @event.Status;
        }

        #endregion
    }
}

