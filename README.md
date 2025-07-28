// Domain/Entities/Process/DelegationEntity.cs
using System;
using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.SeedWork;

namespace BpmBaseApi.Domain.Entities.Process
{
    public class DelegationEntity : BaseJournaledEntity
    {
        public string PrincipalUserCode { get; private set; }
        public string PrincipalUserName { get; private set; }
        public string DeputyUserCode     { get; private set; }
        public string DeputyUserName     { get; private set; }

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
        public void Apply(DelegationCreatedEvent @event)
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
        public void Apply(DelegationRemovedEvent @event) { }

        #endregion
    }
}
