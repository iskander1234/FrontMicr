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
using BpmBaseApi.Domain.Entities.AccessControl;
using BpmBaseApi.Persistence.Configurations.SeedWork;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace BpmBaseApi.Persistence.Configurations.Entities.AccessControl
{
    public class UserConfiguration : BaseEntityConfiguration<UserEntity>
    {
        public override void Configure(EntityTypeBuilder<UserEntity> builder)
        {
            builder.ToTable("User", t => t.HasComment("Данные пользователя"));

            builder.Property(p => p.UserCode)
                   .IsRequired()
                   .HasComment("Код пользователя");

            builder.HasIndex(p => p.UserCode).IsUnique();
            builder.HasAlternateKey(p => p.UserCode);

            builder.Property(p => p.UserName)
                   .HasComment("Имя пользователя");

            builder.HasMany(p => p.UserRoles)
                   .WithOne(ur => ur.User)
                   .HasForeignKey(ur => ur.UserCode)
                   .HasPrincipalKey(u => u.UserCode)
                   .OnDelete(DeleteBehavior.Cascade);

            builder.HasMany(p => p.UserGroups)
                   .WithOne(ug => ug.User)
                   .HasForeignKey(ug => ug.UserCode)
                   .HasPrincipalKey(u => u.UserCode)
                   .OnDelete(DeleteBehavior.Cascade);

            builder.Navigation(p => p.UserRoles).AutoInclude();
            builder.Navigation(p => p.UserGroups).AutoInclude();
        }
    }
}
