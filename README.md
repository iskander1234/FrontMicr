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

