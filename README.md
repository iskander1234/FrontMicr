// Domain/Entities/Process/DelegationEntity.cs
using System;
using BpmBaseApi.Domain.SeedWork;

namespace BpmBaseApi.Domain.Entities.Process
{
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

        // Конструктор и Apply-методы остаются как в примере выше
        public DelegationEntity(string principalCode, string principalName, string deputyCode, string deputyName)
        {
            Id = Guid.NewGuid();
            PrincipalUserCode  = principalCode;
            PrincipalUserName  = principalName;
            DeputyUserCode     = deputyCode;
            DeputyUserName     = deputyName;
        }
    }
}
// Persistence/Configurations/Entities/Process/DelegationConfiguration.cs
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Persistence.Configurations.SeedWork;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace BpmBaseApi.Persistence.Configurations.Entities.Process
{
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
}



dotnet ef migrations add AddDelegationsTable --project BpmBaseApi.Persistence --startup-project BpmBaseApi.WebAPI
dotnet ef database update --project BpmBaseApi.Persistence --startup-project BpmBaseApi.WebAPI
