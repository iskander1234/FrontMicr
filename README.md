// Domain/Entities/Event/Process/DelegationCreatedEvent.cs
using BpmBaseApi.Domain.SeedWork;

namespace BpmBaseApi.Domain.Entities.Event.Process
{
    /// <summary>
    /// Событие: назначен заместитель
    /// </summary>
    public class DelegationCreatedEvent : BaseEntityEvent
    {
        /// <summary>Код пользователя, которого замещают</summary>
        public string PrincipalUserCode { get; set; }

        /// <summary>Имя пользователя, которого замещают</summary>
        public string PrincipalUserName { get; set; }

        /// <summary>Код заместителя</summary>
        public string DeputyUserCode { get; set; }

        /// <summary>Имя заместителя</summary>
        public string DeputyUserName { get; set; }
    }
}


// Domain/Entities/Event/Process/DelegationRemovedEvent.cs
using BpmBaseApi.Domain.SeedWork;

namespace BpmBaseApi.Domain.Entities.Event.Process
{
    /// <summary>
    /// Событие: снят заместитель
    /// </summary>
    public class DelegationRemovedEvent : BaseEntityEvent
    {
        /// <summary>Код пользователя, которого замещали</summary>
        public string PrincipalUserCode { get; set; }

        /// <summary>Код заместителя</summary>
        public string DeputyUserCode { get; set; }
    }
}



// Shared/Commands/Process/CreateDelegationCommand.cs
using MediatR;
using BpmBaseApi.Shared.Dtos;

namespace BpmBaseApi.Shared.Commands.Process
{
    /// <summary>
    /// Назначить заместителя
    /// </summary>
    public class CreateDelegationCommand : IRequest<BaseResponseDto<Guid>>
    {
        /// <summary>Код пользователя, которого замещают</summary>
        public string PrincipalCode { get; set; }

        /// <summary>Код заместителя</summary>
        public string DeputyCode { get; set; }
    }
}


// Shared/Commands/Process/RemoveDelegationCommand.cs
using MediatR;
using BpmBaseApi.Shared.Dtos;

namespace BpmBaseApi.Shared.Commands.Process
{
    /// <summary>
    /// Снять заместителя
    /// </summary>
    public class RemoveDelegationCommand : IRequest<BaseResponseDto<Guid>>
    {
        /// <summary>Код пользователя, которого замещали</summary>
        public string PrincipalCode { get; set; }

        /// <summary>Код заместителя</summary>
        public string DeputyCode { get; set; }
    }
}


// Application/CommandHandlers/Process/CreateDelegationCommandHandler.cs
using AutoMapper;
using MediatR;
using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
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
                    ErrorCodesEnum.NotFound);

            var deputy = await _uow.UserRepository
                .GetByFilterAsync(cancellationToken, u => u.UserCode == command.DeputyCode);
            if (deputy == null)
                throw new HandlerException(
                    $"User '{command.DeputyCode}' not found",
                    ErrorCodesEnum.NotFound);

            // 2. Формируем событие
            var evt = new DelegationCreatedEvent
            {
                EntityId           = Guid.NewGuid(),
                PrincipalUserCode  = principal.UserCode,
                PrincipalUserName  = principal.UserName,
                DeputyUserCode     = deputy.UserCode,
                DeputyUserName     = deputy.UserName
            };

            // 3. Публикуем в репозиторий
            await _uow.DelegationRepository.RaiseEvent(evt, cancellationToken);
            await _uow.CommitAsync(cancellationToken);

            // 4. Возвращаем Id созданной записи
            return new BaseResponseDto<Guid> { Data = evt.EntityId };
        }
    }
}


// Application/CommandHandlers/Process/RemoveDelegationCommandHandler.cs
using MediatR;
using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
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
                    ErrorCodesEnum.NotFound);

            // 2. Формируем событие удаления
            var evt = new DelegationRemovedEvent
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
}


// WebAPI/Controllers/Process/DelegationController.cs
using MediatR;
using Microsoft.AspNetCore.Mvc;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;

namespace BpmBaseApi.WebAPI.Controllers.Process
{
    [ApiController]
    [Route("api/process/delegation")]
    public class DelegationController : ControllerBase
    {
        private readonly IMediator _mediator;

        public DelegationController(IMediator mediator) => _mediator = mediator;

        /// <summary>
        /// Назначить заместителя
        /// </summary>
        [HttpPost("create")]
        [SwaggerResponse(StatusCodes.Status200OK, "Заместитель назначен", typeof(BaseResponseDto<Guid>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        public async Task<IActionResult> CreateAsync(
            [FromBody] CreateDelegationCommand command,
            CancellationToken cancellationToken)
        {
            var result = await _mediator.Send(command, cancellationToken);
            return Ok(result);
        }

        /// <summary>
        /// Снять заместителя
        /// </summary>
        [HttpPost("remove")]
        [SwaggerResponse(StatusCodes.Status200OK, "Заместитель снят", typeof(BaseResponseDto<Guid>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        public async Task<IActionResult> RemoveAsync(
            [FromBody] RemoveDelegationCommand command,
            CancellationToken cancellationToken)
        {
            var result = await _mediator.Send(command, cancellationToken);
            return Ok(result);
        }
    }
}

