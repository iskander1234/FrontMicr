using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Queries.Common;  // ← PagedQuery
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Shared.Queries.Process
{
    public class GetUserRelatedProcessesQuery 
        : PagedQuery, // ← добавили базовый класс с Page/PageSize
          IRequest<BaseResponseDto<List<GetUserRelatedProcessesResponse>>>
    {
        public string UserCode { get; set; } = "";
        public string ProcessCode { get; set; } = "";
        // Page, PageSize приходят из PagedQuery (дефолты 1/50)
    }
}




using AutoMapper;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Dtos.Pagination;           // ★ PagedResponseDto, PaginationMeta
using BpmBaseApi.Shared.Queries.Common;            // ★ Paging.Normalize
using BpmBaseApi.Shared.Queries.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using BpmBaseApi.Application.Common.Extensions;     // ★ ToPage()

namespace BpmBaseApi.Application.QueryHandlers.Process
{
    public class GetUserRelatedProcessesQueryHandler(
        IMapper mapper,
        IUnitOfWork unitOfWork
        ) : IRequestHandler<GetUserRelatedProcessesQuery, BaseResponseDto<List<GetUserRelatedProcessesResponse>>>
    {
        public async Task<BaseResponseDto<List<GetUserRelatedProcessesResponse>>> Handle(
            GetUserRelatedProcessesQuery query, CancellationToken cancellationToken)
        {
            // ★ 1) Нормализуем параметры страницы (дефолты/границы)
            var (page, pageSize) = Paging.Normalize(query.Page, query.PageSize); // ★

            var history = await unitOfWork.ProcessTaskHistoryRepository.GetByFilterListAsync(
                cancellationToken,
                h => h.AssigneeCode == query.UserCode && h.ProcessCode == query.ProcessCode);

            var processDataIds = history
                .Select(h => h.ProcessDataId)
                .Distinct()
                .ToList();

            var processes = await unitOfWork.ProcessDataRepository.GetByFilterListAsync(
                cancellationToken,
                p => processDataIds.Contains(p.Id));

            var mapped = mapper.Map<List<GetUserRelatedProcessesResponse>>(processes);

            // ★ 2) Подсчёт общего и обрезка страницы (in-memory, порядок НЕ меняем)
            var total = mapped.Count;                            // ★
            var pageItems = mapped.ToPage(page, pageSize).ToList(); // ★

            // ★ 3) Возвращаем наследник BaseResponseDto с полем pagination
            return new PagedResponseDto<GetUserRelatedProcessesResponse>  // ★
            {
                Data = pageItems,
                Pagination = new PaginationMeta(total, page, pageSize)
            };
        }
    }
}
