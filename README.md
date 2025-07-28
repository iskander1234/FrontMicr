  public class GetUserTasksQueryHandler(
        IMapper mapper,
        IUnitOfWork unitOfWork,
        IMemoryCache cache
    ) : IRequestHandler<GetUserTasksQuery, BaseResponseDto<List<GetUserTasksResponse>>>
    {
        public async Task<BaseResponseDto<List<GetUserTasksResponse>>> Handle(
            GetUserTasksQuery query,
            CancellationToken cancellationToken)
        {
            string cacheKey = $"tasks:{query.UserCode}";

            // 1) Сначала пробуем взять из кэша
            if (cache.TryGetValue(cacheKey, out List<GetUserTasksResponse> tasksCache))
            {
                return new BaseResponseDto<List<GetUserTasksResponse>> { Data = tasksCache };
            }

            // 2) Получаем список кодов «принципалов», которых я замещаю
            var delegations = await unitOfWork.DelegationRepository
                .GetByDeputyAsync(query.UserCode, cancellationToken);

            var principalCodes = delegations
                .Select(d => d.PrincipalUserCode)
                .Distinct()
                .ToList();

            // 3) Выбираем задачи:
            //    – либо где assignee == я (query.UserCode)
            //    – либо где assignee == любой из principalCodes
            //    и при этом статус == "Pending"
            var tasks = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
                cancellationToken,
                p =>
                    (p.AssigneeCode == query.UserCode ||
                     principalCodes.Contains(p.AssigneeCode))
                    && p.Status == "Pending"
            );

            // 4) Мапим в DTO и сохраняем в кэш на час
            var responseList = mapper.Map<List<GetUserTasksResponse>>(tasks);
            cache.Set(cacheKey, responseList, TimeSpan.FromHours(1));

            // 5) Возвращаем результат
            return new BaseResponseDto<List<GetUserTasksResponse>> { Data = responseList };
        }
    }
