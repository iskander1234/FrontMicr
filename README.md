[HttpPost("getfilesbypd")]
[SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<List<GetFilesByPdResponse>>))]
public async Task<IActionResult> GetFilesByPdAsync([FromBody] GetFilesByPdQuery query, CancellationToken cancellationToken)
{
    var result = await mediator.Send(query, cancellationToken);
    return Ok(result);
}


public class GetFilesByPdQuery : IRequest<BaseResponseDto<List<GetFilesByPdResponse>>>
{
    public Guid ProcessId { get; set; }
    public bool OnlyActive { get; set; } = true; // опционально
}


public class GetFilesByPdQueryHandler(
    IMapper mapper,
    IUnitOfWork unitOfWork
) : IRequestHandler<GetFilesByPdQuery, BaseResponseDto<List<GetFilesByPdResponse>>>
{
    public async Task<BaseResponseDto<List<GetFilesByPdResponse>>> Handle(GetFilesByPdQuery query, CancellationToken cancellationToken)
    {
        // Проверим, что PD существует (можно убрать, если не нужно)
        var pd = await unitOfWork.ProcessDataRepository.GetByFilterAsync(
            cancellationToken, p => p.Id == query.ProcessId
        ) ?? throw new HandlerException("Запрос с таким идентификатором не найден", ErrorCodesEnum.Business);

        var files = await unitOfWork.ProcessFileRepository.GetByFilterListAsync(
            cancellationToken,
            f => f.ProcessDataId == query.ProcessId
                 && (!query.OnlyActive || f.State == ProcessFileState.Active)
        );

        // Если нужно – отсортируем, например, по дате создания
        // files = files.OrderBy(f => f.Created).ToList();

        var data = mapper.Map<List<GetFilesByPdResponse>>(files);
        return new BaseResponseDto<List<GetFilesByPdResponse>> { Data = data };
    }
}


public class GetFilesByPdResponse
{
    public Guid   FileId   { get; set; }
    public string FileName { get; set; } = default!;
    public string FileType { get; set; } = default!;
}
