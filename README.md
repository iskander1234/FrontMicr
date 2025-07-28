public class RemoveDelegationCommandHandler
    : IRequestHandler<RemoveDelegationCommand, BaseResponseDto<Guid>>
{
    private readonly IUnitOfWork _uow;

    public RemoveDelegationCommandHandler(IUnitOfWork uow) => _uow = uow;

    public async Task<BaseResponseDto<Guid>> Handle(
        RemoveDelegationCommand command,
        CancellationToken cancellationToken)
    {
        // 1) Ищем существующую делегацию
        var list = await _uow.DelegationRepository
            .GetByFilterListAsync(cancellationToken, d =>
                d.PrincipalUserCode == command.PrincipalCode &&
                d.DeputyUserCode    == command.DeputyCode);

        var entity = list.FirstOrDefault();
        if (entity == null)
            throw new HandlerException(
                "Делегация не найдена",
                ErrorCodesEnum.Business);

        // 2) Удаляем запись напрямую
        await _uow.DelegationRepository.DeleteByIdAsync(entity.Id, cancellationToken);
        await _uow.CommitAsync(cancellationToken);

        // 3) Возвращаем Id удалённой записи
        return new BaseResponseDto<Guid> { Data = entity.Id };
    }
}
