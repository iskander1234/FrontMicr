using BpmBaseApi.Domain.Entities.Event.Process;
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

    // EF + репозиторий по new() требуют public parameterless
    public DelegationEntity() { }

    // Бизнес-конструктор
    public DelegationEntity(
        string principalCode,
        string principalName,
        string deputyCode,
        string deputyName)
    {
        Id                 = Guid.NewGuid();
        PrincipalUserCode  = principalCode;
        PrincipalUserName  = principalName;
        DeputyUserCode     = deputyCode;
        DeputyUserName     = deputyName;
    }

    #region Apply

    // Этот метод будет вызван внутри RaiseEvent(evt)
    public void Apply(ProcessDelegationCreatedEvent @event)
    {
        // Заполняем все четыре поля из события
        Id                 = @event.EntityId;
        PrincipalUserCode  = @event.PrincipalUserCode;
        PrincipalUserName  = @event.PrincipalUserName;
        DeputyUserCode     = @event.DeputyUserCode;
        DeputyUserName     = @event.DeputyUserName;
    }

    // При удалении делегации можно ничего не делать —
    // репозиторий сам выполнит DeleteByIdAsync
    public void Apply(ProcessDelegationRemovedEvent @event)
    {
        PrincipalUserCode = @event.PrincipalUserCode;
        DeputyUserCode = @event.DeputyUserCode;
    }

    #endregion
}

