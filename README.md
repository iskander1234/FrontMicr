using BpmBaseApi.Domain.Entities.Event.AccessControl;
using BpmBaseApi.Domain.SeedWork;

namespace BpmBaseApi.Domain.Entities.AccessControl
{
    public class UserEntity : BaseJournaledEntity
    {
        /// <summary>
        /// Код пользователя
        /// </summary>
        public string UserCode { get; private set; }

        /// <summary>
        /// Имя пользователя
        /// </summary>
        public string UserName { get; private set; }

        /// <summary>
        /// Коллекция ролей, назначенных пользователю.
        /// </summary>
        public virtual List<UserRoleEntity> UserRoles { get; private set; } = [];

        /// <summary>
        /// Коллекция групп, в которых состоит пользователь.
        /// </summary>
        public virtual List<UserGroupEntity> UserGroups { get; private set; } = [];

        #region Apply

        public void Apply(UserCreatedEvent @event)
        {
            Id = Guid.NewGuid();
            @event.EntityId = Id;
            UserCode = @event.UserCode;
            UserName = @event.UserName;
        }

        #endregion

    }
}

using BpmBaseApi.Domain.SeedWork;

namespace BpmBaseApi.Domain.Entities.Event.AccessControl
{
    public class UserCreatedEvent : BaseEntityEvent
    {
        /// <summary>
        /// Код пользователя
        /// </summary>
        public string UserCode { get; set; }
        /// <summary>
        /// Имя пользователя
        /// </summary>
        public string UserName { get; set; }
    }
}

using BpmBaseApi.Shared.Dtos;
using MediatR;

namespace BpmBaseApi.Shared.Commands.AccessControl
{
    public class CreateUserCommand : IRequest<BaseResponseDto<Guid>>
    {
        /// <summary>
        /// Код пользователя
        /// </summary>
        public string UserCode { get; set; }

        /// <summary>
        /// Имя пользователя
        /// </summary>
        public string UserName { get; set; }
    }
}


using AutoMapper;
using BpmBaseApi.Domain.Entities.Event.AccessControl;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Commands.AccessControl;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Queries;
using BpmBaseApi.Shared.Responses;
using MediatR;

namespace BpmBaseApi.Application.CommandHandlers.AccessControl
{
    public class CreateUserCommandHandler(IMapper mapper,
        IUnitOfWork unitOfWork) : IRequestHandler<CreateUserCommand, BaseResponseDto<Guid>>
    {
        public async Task<BaseResponseDto<Guid>> Handle(CreateUserCommand command, CancellationToken cancellationToken)
        {
            var userData = await unitOfWork.UserRepository.GetByFilterAsync(cancellationToken, p => p.UserCode.ToLower() == command.UserCode.ToLower());

            if (userData != null)
            {
                throw new HandlerException("Пользователь с таким логином уже есть в системе", ErrorCodesEnum.Business);
            }

            var userCreatedEvent = new UserCreatedEvent
            {
                UserCode = command.UserCode.ToLower(),
                UserName = command.UserName
            };

            await unitOfWork.UserRepository.RaiseEvent(userCreatedEvent, cancellationToken);

            return new BaseResponseDto<Guid> { Data = userCreatedEvent.EntityId };
        }
    }
}
/// <summary>
        /// Добавление пользователя
        /// </summary>
        /// <param name="query">Запрос для добавления пользователя</param>
        /// <param name="cancellationToken">Маркер отмены, используемый для отмены HTTP-запроса.</param>
        /// <returns></returns>
        [HttpPost("createuser")]
        [SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<Guid>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        [SwaggerResponse(StatusCodes.Status503ServiceUnavailable)]
        public async Task<IActionResult> CreateUserAsync([FromBody] CreateUserCommand command, CancellationToken cancellationToken)
        {
            var result = await mediator.Send(command, cancellationToken);

            return Ok(result);
        }

что у меня реализованно 

using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Persistence.Configurations.SeedWork;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace BpmBaseApi.Persistence.Configurations.Entities.Process;

/// <summary>
/// Настройка таблицы Delegations для сущности DelegationEntity
/// </summary>
public class DelegationConfiguration : BaseEntityConfiguration<DelegationEntity>
{
    public override void Configure(EntityTypeBuilder<DelegationEntity> builder)
    {
        // имя таблицы и комментарий
        builder.ToTable("Delegations", t => t.HasComment("Таблица заместителей"));

        // ключевой столбец Id приходит из BaseJournaledEntity

        // PrincipalUserCode — обязательное поле
        builder.Property(p => p.PrincipalUserCode)
            .IsRequired()
            .HasMaxLength(64)
            .HasComment("Код пользователя, которого замещают");

        // PrincipalUserName — не-null, можно хранить до 256 символов
        builder.Property(p => p.PrincipalUserName)
            .IsRequired()
            .HasMaxLength(256)
            .HasComment("Имя пользователя, которого замещают");

        // DeputyUserCode — обязательное поле
        builder.Property(p => p.DeputyUserCode)
            .IsRequired()
            .HasMaxLength(64)
            .HasComment("Код заместителя");

        // DeputyUserName — не-null, можно хранить до 256 символов
        builder.Property(p => p.DeputyUserName)
            .IsRequired()
            .HasMaxLength(256)
            .HasComment("Имя заместителя");

        // Уникальный индекс, чтобы не дублировать одну и ту же пару
        builder.HasIndex(p => new { p.PrincipalUserCode, p.DeputyUserCode })
            .IsUnique()
            .HasDatabaseName("UX_Delegations_Principal_Deputy");

        base.Configure(builder);
    }
}
using BpmBaseApi.Domain.SeedWork;

namespace BpmBaseApi.Domain.Entities.Process;

/// <summary>
/// Делегирование: кто замещает кого
/// </summary>
public class DelegationEntity : BaseJournaledEntity
{
    /// <summary>Код пользователя, которого замещают</summary>
    public string PrincipalUserCode { get; private set; }

    /// <summary>Имя пользователя, которого замещают</summary>
    public string PrincipalUserName { get; private set; }

    /// <summary>Код заместителя</summary>
    public string DeputyUserCode { get; private set; }

    /// <summary>Имя заместителя</summary>
    public string DeputyUserName { get; private set; }

    // 1) Пустой (parameterless) конструктор нужен EF Core для materialization.
    protected DelegationEntity() { }

    /// <summary>
    /// 2) Бизнес-конструктор для создания нового делегирования.
    /// </summary>
    /// <param name="principalCode">Код принципала</param>
    /// <param name="principalName">Имя принципала</param>
    /// <param name="deputyCode">Код заместителя</param>
    /// <param name="deputyName">Имя заместителя</param>
    public DelegationEntity(
        string principalCode,
        string principalName,
        string deputyCode,
        string deputyName)
    {
        Id = Guid.NewGuid();
        PrincipalUserCode = principalCode;
        PrincipalUserName = principalName;
        DeputyUserCode = deputyCode;
        DeputyUserName = deputyName;
    }
}


