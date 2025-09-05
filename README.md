 /// <summary>Список черновиков по InitiatorCode</summary>
    [HttpGet("drafts")]
    [SwaggerResponse(StatusCodes.Status200OK, "OK", typeof(BaseResponseDto<List<DraftItemResponse>>))]
    public async Task<IActionResult> GetDrafts([FromQuery] string initiatorCode, CancellationToken ct)
    {
        var result = await _mediator.Send(new GetDraftsByInitiatorQuery(initiatorCode), ct);
        return Ok(result);
    }
    
    using MediatR;
using BpmBaseApi.Shared.Dtos;

namespace BpmBaseApi.Shared.Queries.Process
{
    public record GetDraftsByInitiatorQuery(string InitiatorCode)
        : IRequest<BaseResponseDto<List<DraftItemResponse>>>;

    public class DraftItemResponse
    {
        public Guid   ProcessGuid { get; set; }
        public string RegNumber   { get; set; } = "";
        public string Title       { get; set; } = "";
        public string ProcessCode { get; set; } = "";
        public string ProcessName { get; set; } = "";
        public string StatusCode  { get; set; } = "Draft";
        public string StatusName  { get; set; } = "Черновик";
        public DateTime Created   { get; set; }
        public DateTime? Modified { get; set; }
    }
}


using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Queries.Process;
using MediatR;

namespace BpmBaseApi.Application.QueryHandlers.Process
{
    public class GetDraftsByInitiatorQueryHandler(
        IUnitOfWork unitOfWork
    ) : IRequestHandler<GetDraftsByInitiatorQuery, BaseResponseDto<List<DraftItemResponse>>>
    {
        public async Task<BaseResponseDto<List<DraftItemResponse>>> Handle(GetDraftsByInitiatorQuery query, CancellationToken ct)
        {
            if (string.IsNullOrWhiteSpace(query.InitiatorCode))
                throw new HandlerException("InitiatorCode обязателен", ErrorCodesEnum.Business);

            var initiator = query.InitiatorCode.Trim().ToLowerInvariant();

            var drafts = await unitOfWork.ProcessDataRepository.GetByFilterListAsync(
                ct,
                p => p.InitiatorCode != null
                     && p.InitiatorCode.ToLower() == initiator
                     && p.StatusCode == "Draft"
            );

            var result = drafts
                .OrderByDescending(p => p.Created)
                .Select(p => new DraftItemResponse
                {
                    ProcessGuid = p.Id,
                    RegNumber   = p.RegNumber,
                    Title       = p.Title,
                    ProcessCode = p.ProcessCode,
                    ProcessName = p.ProcessName,
                    StatusCode  = p.StatusCode,
                    StatusName  = p.StatusName,
                    Created     = p.Created,
                    Modified    = p.Modified
                })
                .ToList();

            return new BaseResponseDto<List<DraftItemResponse>> { Data = result };
        }
    }
}
