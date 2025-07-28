using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using MediatR;

namespace BpmBaseApi.Application.CommandHandlers.Process;

public class CreateDelegationCommandHandler
    : IRequestHandler<CreateDelegationCommand, BaseResponseDto<Guid>>
{
    private readonly IUnitOfWork _uow;

    public CreateDelegationCommandHandler(IUnitOfWork uow) => _uow = uow;

    public async Task<BaseResponseDto<Guid>> Handle(
        CreateDelegationCommand command,
        CancellationToken cancellationToken)
    {
        // 1. Получаем имена из UserEntity
        var principal = await _uow.UserRepository
            .GetByFilterAsync(cancellationToken, u => u.UserCode == command.PrincipalCode);
        if (principal == null)
            throw new HandlerException(
                $"User '{command.PrincipalCode}' not found",
                ErrorCodesEnum.Business);

        var deputy = await _uow.UserRepository
            .GetByFilterAsync(cancellationToken, u => u.UserCode == command.DeputyCode);
        if (deputy == null)
            throw new HandlerException(
                $"User '{command.DeputyCode}' not found",
                ErrorCodesEnum.Business);

        // 2. Формируем событие
        var evt = new ProcessDelegationCreatedEvent
        {
            EntityId = Guid.NewGuid(),
            PrincipalUserCode = principal.UserCode,
            PrincipalUserName = principal.UserName,
            DeputyUserCode = deputy.UserCode,
            DeputyUserName = deputy.UserName
        };

        // 3. Публикуем в репозиторий
        await _uow.DelegationRepository.RaiseEvent(evt, cancellationToken);
        await _uow.CommitAsync(cancellationToken);

        // 4. Возвращаем Id созданной записи
        return new BaseResponseDto<Guid> { Data = evt.EntityId };
    }
}



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
