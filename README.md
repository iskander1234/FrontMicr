using Microsoft.EntityFrameworkCore;

namespace DinDin.Models;

public partial class BpmcoreContext : DbContext
{
    public BpmcoreContext()
    {
    }

    public BpmcoreContext(DbContextOptions<BpmcoreContext> options)
        : base(options)
    {
    }

    public virtual DbSet<Department> Departments { get; set; }

    public virtual DbSet<Employee> Employees { get; set; }
    public DbSet<LDAPEmployee> LdapEmployees => Set<LDAPEmployee>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Конфигурация из самого LDAPEmployee
        modelBuilder.ApplyConfiguration(new LDAPEmployee());
    }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        => optionsBuilder.UseNpgsql("Host=localhost;Port=5432;Database=bpmbase7;Username=postgres;Password=postgres")
                        .EnableSensitiveDataLogging();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.HasPostgresExtension("csd", "pgcrypto");

        modelBuilder.Entity<Department>(entity =>
        {
            entity.HasKey(e => e.Id).HasName("departments2_pkey");

            entity.ToTable("Departments", "public");

            entity.Property(e => e.Id).HasColumnName("id");
            entity.Property(e => e.Actual).HasColumnName("actual");
            entity.Property(e => e.ManagerId).HasColumnName("manager_id");
            entity.Property(e => e.Name).HasColumnName("name");
            entity.Property(e => e.ParentId).HasColumnName("parent_id");

            entity.HasOne(d => d.Parent).WithMany(p => p.InverseParent)
                .HasForeignKey(d => d.ParentId)
                .OnDelete(DeleteBehavior.SetNull)
                .HasConstraintName("departments2_parent_id_fkey");
        });

        modelBuilder.Entity<Employee>(entity =>
        {
            entity.HasKey(e => e.Id).HasName("employees_pkey");

            entity.ToTable("Employees", "public");

            entity.HasIndex(e => e.TabNumber, "employees_tab_number_key").IsUnique();

            entity.Property(e => e.Id)
                .UseIdentityColumn() // если ты хочешь оставить sequence
                .HasColumnName("id");

            entity.Property(e => e.DepId).HasColumnName("dep_id");
            entity.Property(e => e.DepName).HasColumnName("dep_name");
            entity.Property(e => e.Disabled).HasColumnName("disabled");
            entity.Property(e => e.IsFilial).HasColumnName("is_filial");
            entity.Property(e => e.IsManager).HasColumnName("is_manager");
            entity.Property(e => e.LocalPhone).HasColumnName("local_phone");
            entity.Property(e => e.Login).HasColumnName("login");
            entity.Property(e => e.Mail).HasColumnName("mail");
            entity.Property(e => e.ManagerTabNumber).HasColumnName("manager_tab_number");
            entity.Property(e => e.MobilePhone).HasColumnName("mobile_phone");
            entity.Property(e => e.Name).HasColumnName("name");
            entity.Property(e => e.Position).HasColumnName("position");
            entity.Property(e => e.StatusCode).HasColumnName("status_code");
            entity.Property(e => e.StatusDescription).HasColumnName("status_description");
            entity.Property(e => e.TabNumber).HasColumnName("tab_number");
            entity.Property(e => e.ParentDepId).HasColumnName("parent_dep_id");
            entity.Property(e => e.ParentDepName).HasColumnName("parent_dep_name");

            entity.HasOne(d => d.ManagerTabNumberNavigation).WithMany(p => p.InverseManagerTabNumberNavigation)
                .HasPrincipalKey(p => p.TabNumber)
                .HasForeignKey(d => d.ManagerTabNumber)
                .OnDelete(DeleteBehavior.SetNull)
                .HasConstraintName("employees_manager_tab_number_fkey");
        });

        OnModelCreatingPartial(modelBuilder);
    }

    
    partial void OnModelCreatingPartial(ModelBuilder modelBuilder);
}
