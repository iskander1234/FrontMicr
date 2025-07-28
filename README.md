using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using MediatR;

namespace BpmBaseApi.Application.CommandHandlers.Process;

public class RemoveDelegationCommandHandler
    : IRequestHandler<RemoveDelegationCommand, BaseResponseDto<Guid>>
{
    private readonly IUnitOfWork _uow;

    public RemoveDelegationCommandHandler(IUnitOfWork uow) => _uow = uow;

    public async Task<BaseResponseDto<Guid>> Handle(
        RemoveDelegationCommand command,
        CancellationToken cancellationToken)
    {
        // 1. Находим существующую делегацию
        var list = await _uow.DelegationRepository
            .GetByFilterListAsync(cancellationToken, d =>
                d.PrincipalUserCode == command.PrincipalCode &&
                d.DeputyUserCode    == command.DeputyCode);

        var entity = list.FirstOrDefault();
        if (entity == null)
            throw new HandlerException(
                "Делегация не найдена",
                ErrorCodesEnum.Business);

        // 2. Формируем событие удаления
        var evt = new ProcessDelegationRemovedEvent
        {
            EntityId           = entity.Id,
            PrincipalUserCode  = command.PrincipalCode,
            DeputyUserCode     = command.DeputyCode
        };

        // 3. Публикуем и удаляем
        await _uow.DelegationRepository.RaiseEvent(evt, cancellationToken);
        await _uow.CommitAsync(cancellationToken);

        return new BaseResponseDto<Guid> { Data = evt.EntityId };
    }
}
