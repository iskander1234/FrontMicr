using BpmBaseApi.Domain.Entities.Event.Employees;
using BpmBaseApi.Domain.SeedWork;

namespace BpmBaseApi.Domain.Entities.Employees;

/// <summary>
/// Сотрудник
/// </summary>
public class EmployeeEntity : BaseIntEntity
{
    public string Name { get; set; }
    public string? Position { get; set; }
    public string? Login { get; set; }
    public string? Mail { get; set; }
    public string? TabNumber { get; set; }
    public string? LocalPhone { get; set; }
    public string? MobilePhone { get; set; }
    public string? DepId { get; set; }
    public string? DepName { get; set; }
    public string? ParentDepId { get; set; }
    public string? ParentDepName { get; set; }
    public string? LoginAd { get; set; }
    public string? ManagerTabNumber { get; set; }
    public int? StatusCode { get; set; }
    public string? StatusDescription { get; set; }

    public bool Disabled { get; set; }
    public bool IsManager { get; set; }
    public bool IsFilial { get; set; }

    public virtual EmployeeEntity? ManagerTabNumberNavigation { get; set; }
    public virtual ICollection<EmployeeEntity> InverseManagerTabNumberNavigation { get; set; } = new List<EmployeeEntity>();

    public void Apply(EmployeeCreatedEvent @event)
    {
        Name = @event.Name;
        Position = @event.Position;
        Login = @event.Login;
        LoginAd = @event.LoginAd;
        StatusCode = @event.StatusCode;
        StatusDescription = @event.StatusDescription;
        DepId = @event.DepId;
        DepName = @event.DepName;
        IsFilial = @event.IsFilial;
        Mail = @event.Mail;
        LocalPhone = @event.LocalPhone;
        MobilePhone = @event.MobilePhone;
        IsManager = @event.IsManager;
        ManagerTabNumber = @event.ManagerTabNumber;
        Disabled = @event.Disabled;
        TabNumber = @event.TabNumber;
        ParentDepId = @event.ParentDepId;
        ParentDepName = @event.ParentDepName;
    }

}

using BpmBaseApi.Domain.SeedWork;

namespace BpmBaseApi.Domain.Entities.Event.Employees;

public class EmployeeCreatedEvent : BaseEntityEvent
{
    public int EntityId { get; set; }
    public string? Name { get; set; }
    public string? Position { get; set; }
    public string? Login { get; set; }
    public string? LoginAd { get; set; }
    public int? StatusCode { get; set; }
    public string? StatusDescription { get; set; }
    public string? DepId { get; set; }
    public string? DepName { get; set; }
    public bool IsFilial { get; set; }
    public string? Mail { get; set; }
    public string? LocalPhone { get; set; }
    public string? MobilePhone { get; set; }
    public bool IsManager { get; set; }
    public string? ManagerTabNumber { get; set; }
    public bool Disabled { get; set; }
    public string? TabNumber { get; set; }
    public string? ParentDepId { get; set; }
    public string? ParentDepName { get; set; }
}

using BpmBaseApi.Domain.Entities.Employees;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace BpmBaseApi.Persistence.Configurations.Entities.Employees;

public class EmployeeConfiguration : IEntityTypeConfiguration<EmployeeEntity>
{
    public void Configure(EntityTypeBuilder<EmployeeEntity> entity)
    {
        entity.ToTable("Employees", "public");

        entity.HasKey(e => e.Id).HasName("employees_pkey");

        entity.HasIndex(e => e.TabNumber).IsUnique().HasDatabaseName("employees_tab_number_key");

        entity.Property(e => e.Id)
            .UseIdentityColumn() // если ты хочешь оставить sequence
            .HasColumnName("id");

        entity.Property(e => e.Name).HasColumnName("name");
        entity.Property(e => e.Position).HasColumnName("position");
        entity.Property(e => e.Login).HasColumnName("login");
        entity.Property(e => e.LoginAd).HasColumnName("login_ad");
        entity.Property(e => e.Mail).HasColumnName("mail");
        entity.Property(e => e.TabNumber).HasColumnName("tab_number");
        entity.Property(e => e.LocalPhone).HasColumnName("local_phone");
        entity.Property(e => e.MobilePhone).HasColumnName("mobile_phone");
        entity.Property(e => e.DepId).HasColumnName("dep_id");
        entity.Property(e => e.DepName).HasColumnName("dep_name");
        entity.Property(e => e.ParentDepId).HasColumnName("parent_dep_id");
        entity.Property(e => e.ParentDepName).HasColumnName("parent_dep_name");
        entity.Property(e => e.ManagerTabNumber).HasColumnName("manager_tab_number");
        entity.Property(e => e.StatusCode).HasColumnName("status_code");
        entity.Property(e => e.StatusDescription).HasColumnName("status_description");
        entity.Property(e => e.Disabled).HasColumnName("disabled");
        entity.Property(e => e.IsManager).HasColumnName("is_manager");
        entity.Property(e => e.IsFilial).HasColumnName("is_filial");

        entity.HasOne(e => e.ManagerTabNumberNavigation)
            .WithMany(e => e.InverseManagerTabNumberNavigation)
            .HasPrincipalKey(e => e.TabNumber)
            .HasForeignKey(e => e.ManagerTabNumber)
            .OnDelete(DeleteBehavior.SetNull)
            .HasConstraintName("employees_manager_tab_number_fkey");
    }
}
