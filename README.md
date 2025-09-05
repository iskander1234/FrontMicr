using MediatR;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;

namespace BpmBaseApi.Shared.Queries.Process
{
    // фильтр: инициатор обязателен; processCode/пагинация — опционально
    public record GetDraftsByInitiatorQuery(
        string InitiatorCode,
        string? ProcessCode = null,
        int? Skip = null,
        int? Take = null
    ) : IRequest<BaseResponseDto<List<GetDraftListItemResponse>>>;

    public class GetDraftListItemResponse
    {
        public Guid ProcessGuid { get; set; }
        public string RegNumber { get; set; } = "";
        public string Title { get; set; } = "";
        public string ProcessCode { get; set; } = "";
        public string ProcessName { get; set; } = "";
        public string StatusCode { get; set; } = "Draft";
        public string StatusName { get; set; } = "Черновик";
        public DateTime Created { get; set; }
        public DateTime? Modified { get; set; }
    }
}



using AutoMapper;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Queries.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Application.QueryHandlers.Process
{
    public class GetDraftsByInitiatorQueryHandler(
        IUnitOfWork unitOfWork,
        IMapper mapper
    ) : IRequestHandler<GetDraftsByInitiatorQuery, BaseResponseDto<List<GetDraftListItemResponse>>>
    {
        public async Task<BaseResponseDto<List<GetDraftListItemResponse>>> Handle(GetDraftsByInitiatorQuery query, CancellationToken ct)
        {
            if (string.IsNullOrWhiteSpace(query.InitiatorCode))
                throw new HandlerException("InitiatorCode обязателен", error: null);

            var initiatorLower = query.InitiatorCode.Trim().ToLowerInvariant();
            var processCode    = query.ProcessCode;

            // Забираем все Draft'ы инициатора (и опционально по коду процесса)
            var drafts = await unitOfWork.ProcessDataRepository.GetByFilterListAsync(
                ct,
                p => p.InitiatorCode != null
                     && p.InitiatorCode.ToLower() == initiatorLower
                     && p.StatusCode == "Draft"
                     && (processCode == null || p.ProcessCode == processCode)
            );

            // Сортировка: новые сверху
            var ordered = drafts
                .OrderByDescending(p => p.Created)
                .ToList();

            // Пагинация (если заданы Skip/Take)
            if (query.Skip is int s && s > 0)
                ordered = ordered.Skip(s).ToList();
            if (query.Take is int t && t > 0)
                ordered = ordered.Take(t).ToList();

            // Маппинг → лёгкий DTO
            // Можно через AutoMapper, если настроен профиль. Ниже — ручной, самодостаточный.
            var result = ordered.Select(p => new GetDraftListItemResponse
            {
                ProcessGuid = p.Id,
                RegNumber   = p.RegNumber,
                Title       = p.Title,
                ProcessCode = p.ProcessCode,
                ProcessName = p.ProcessName,
                StatusCode  = p.StatusCode,       // ожидается "Draft"
                StatusName  = p.StatusName,       // ожидается "Черновик"
                Created     = p.Created,
                Modified    = p.Modified
            }).ToList();

            return new BaseResponseDto<List<GetDraftListItemResponse>> { Data = result };
        }
    }
}
