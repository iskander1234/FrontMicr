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

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        => optionsBuilder.UseNpgsql("Host=localhost;Port=5432;Database=bpmbase6;Username=postgres;Password=postgres")
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


namespace DinDin.Models;

public partial class Department
{
    public string Id { get; set; } = null!;

    public string? Name { get; set; }

    public int Actual { get; set; }

    public string? ManagerId { get; set; }

    public string? ParentId { get; set; }

    public virtual List<Department> InverseParent { get; set; } = new List<Department>();

    public virtual Department? Parent { get; set; }
}

namespace DinDin.Models;

public partial class Employee
{
    public int Id { get; set; }

    public string? Name { get; set; }

    public string? Position { get; set; }

    public string? Login { get; set; }

    public int? StatusCode { get; set; }

    public string? StatusDescription { get; set; }

    public string? DepId { get; set; }

    public string? DepName { get; set; }

    public string? ParentDepId { get; set; }

    public string? ParentDepName { get; set; }

    public bool? IsFilial { get; set; }

    public string? Mail { get; set; }

    public string? LocalPhone { get; set; }

    public string? MobilePhone { get; set; }

    public bool? IsManager { get; set; }

    public string? ManagerTabNumber { get; set; }

    public bool? Disabled { get; set; }

    public string TabNumber { get; set; } = null!;

    public virtual ICollection<Employee> InverseManagerTabNumberNavigation { get; set; } = new List<Employee>();

    public virtual Employee? ManagerTabNumberNavigation { get; set; }
}


using DinDin.Models;
using EFCore.BulkExtensions;

namespace DinDin.Repositories
{
    internal class DepartmentRepository(BpmcoreContext userContext)// : IListRepository<Department>
    {
        public async Task Update(List<Department> collection)
        {
            await userContext.BulkInsertOrUpdateAsync(collection, options =>
            {
                options.BatchSize = 1000;
                options.IncludeGraph = true;
            });
        }
    }
}

using DinDin.Models;
using EFCore.BulkExtensions;
using Microsoft.EntityFrameworkCore;

namespace DinDin.Repositories
{
    internal class EmployeeRepository(BpmcoreContext context)
    {
        public async Task Update(List<Employee> collection)
        {
            await context.BulkInsertOrUpdateAsync(collection, options =>
            {
                options.BatchSize = 1000;
                options.IncludeGraph = true;
                options.SetOutputIdentity = true;
                //options.UpdateByProperties = ["Login"];
            });
        }

        public async Task Update2(List<Employee> employees)
        {
            var bdEmployees = await context.Employees.ToListAsync();
            //var bdDeps = await context.Departments.ToListAsync();

            bdEmployees.ForEach(e =>
            {
                e.StatusCode = -1;
                e.StatusDescription = "Уволен";
            });

            foreach (var employee in employees)
            {
                var oldEmployee = bdEmployees.FirstOrDefault(a => a.TabNumber == employee.TabNumber);

                if (oldEmployee == null)
                    context.Employees.Add(employee);
                else
                {
                    oldEmployee.Name = employee.Name;
                    oldEmployee.Position = employee.Position;
                    oldEmployee.Login = employee.Login;
                    oldEmployee.StatusCode = employee.StatusCode;
                    oldEmployee.StatusDescription = employee.StatusDescription;
                    oldEmployee.DepId = employee.DepId;
                    oldEmployee.DepName = employee.DepName;
                    //oldEmployee.ParentDepId = bdDeps.FirstOrDefault(a => a.Id == employee.DepId)?.ParentId;
                    //oldEmployee.ParentDepName = bdDeps.FirstOrDefault(a => a.Id == oldEmployee.ParentDepId)?.Name;
                    oldEmployee.ParentDepId = employee.ParentDepId;
                    oldEmployee.ParentDepName = employee.ParentDepName;
                    oldEmployee.IsFilial = employee.IsFilial;
                    oldEmployee.Mail = employee.Mail;
                    oldEmployee.LocalPhone = employee.LocalPhone;
                    oldEmployee.MobilePhone = employee.MobilePhone;
                    oldEmployee.IsManager = employee.IsManager;
                    oldEmployee.ManagerTabNumber = employee.ManagerTabNumber;
                    oldEmployee.Disabled = employee.Disabled;
                    oldEmployee.TabNumber = employee.TabNumber;
                }
            }

            await context.SaveChangesAsync();
        }
    }
}

using DinDin.Models;
using DinDin.Models.Dto;
using DinDin.Repositories;
using DinDin.Services;
using Mapster;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using System;

namespace DinDin;

class Program
{
    private static IConfiguration Configuration;
    private static List<Department> _deps = new List<Department>();

    static async Task Main()
    {
        MapsterConfig();
        Configuration = BuildConfiguration();
        var serviceProvider = ConfigureServices(Configuration);

        await StartData(serviceProvider);
    }

    private static async Task StartData(ServiceProvider serviceProvider)
    {
        var oneCService = serviceProvider.GetRequiredService<OneCService>();

        var deps = (await oneCService.GetDepartments()).Adapt<List<Department>>();
        await serviceProvider.GetRequiredService<DepartmentRepository>().Update(deps);
        //PopulateParentId(deps);

        var employees = (await oneCService.GetEmployees()).Adapt<List<Employee>>();

        foreach (var employee in employees)
        {
            var e = employee;
            if (e.Name.Contains("Аристом"))
                e = employee;

            foreach (var branch in deps)
            {
                foreach (var department in branch.InverseParent)
                {
                    if (employee.DepId == department.Id)
                    {
                        employee.ParentDepId = department.Id;
                        employee.ParentDepName = department.Name;
                        break;
                    }

                    foreach (var division in department.InverseParent)
                    {
                        if (employee.DepId == division.Id)
                        {
                            if (branch.Id == "01.001000")
                            {
                                employee.ParentDepId = department.Id;
                                employee.ParentDepName = department.Name;
                                break;
                            }

                            employee.ParentDepId = branch.Id;
                            employee.ParentDepName = branch.Name;
                            break;
                        }
                    }
                }
            }
        }

        await serviceProvider.GetRequiredService<EmployeeRepository>().Update2(employees);
    }

    private static void MapsterConfig()
    {
        TypeAdapterConfig<EmployeeDTO, Employee>
            .NewConfig()
            .Map(dest => dest.IsFilial, src => src.IsFilial == "1")
            .Map(dest => dest.IsManager, src => src.IsManager == "1")
            .Map(dest => dest.Disabled, src => src.Disabled == "1")
            .Map(dest => dest.ManagerTabNumber, src => src.ManagerId);

        TypeAdapterConfig<DepartmentDTO, Department>
            .NewConfig()
            .Map(dest => dest.InverseParent, src => src.Children);
    }

    private static ServiceProvider ConfigureServices(IConfiguration configuration)
    {
        var connectionString = configuration.GetConnectionString("BpmCore") ?? throw new Exception("ConnectionString BpmCore was not found");

        return new ServiceCollection()
            .AddDbContext<BpmcoreContext>(options => options.UseNpgsql(connectionString))
            .AddSingleton<OneCService>()
            .AddSingleton<DepartmentRepository>()
            .AddSingleton<EmployeeRepository>()
            .BuildServiceProvider();
    }

    private static IConfiguration BuildConfiguration()
    {
        var environment = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") ?? "Production";

        return new ConfigurationBuilder()
            .SetBasePath(Directory.GetCurrentDirectory())
            .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
            .Build();
    }

    private static void PopulateParentId(List<Department> departments, string? parentId = null)
    {
        foreach (var department in departments)
        {
            _deps.Add(department);
            PopulateParentId(department.InverseParent, department.Id);
        }
    }
}

using System.Text.Json.Serialization;

namespace DinDin.Models.Dto
{
    internal class DepartmentDTO
    {
        [JsonPropertyName("id")]
        public string? Id { get; set; }

        [JsonPropertyName("name")]
        public string? Name { get; set; }

        [JsonPropertyName("actual")]
        public int Actual { get; set; }

        [JsonPropertyName("manager_id")]
        public string? ManagerId { get; set; }

        [JsonPropertyName("departments")]
        public List<DepartmentDTO>? Children { get; set; }
    }
}

using System.Text.Json.Serialization;

namespace DinDin.Models
{
    internal class EmployeeDTO
    {
        [JsonPropertyName("name")]
        public string? Name { get; set; }

        [JsonPropertyName("position")]
        public string? Position { get; set; }

        [JsonPropertyName("login")]
        public string? Login { get; set; }

        [JsonPropertyName("status_code")]
        public int StatusCode { get; set; }

        [JsonPropertyName("status_description")]
        public string? StatusDescription { get; set; }

        [JsonPropertyName("dep_id")]
        public string? DepId { get; set; }

        [JsonPropertyName("dep_name")]
        public string? DepName { get; set; }

        [JsonPropertyName("is_filial")]
        public string? IsFilial { get; set; }

        [JsonPropertyName("mail")]
        public string? Mail { get; set; }

        [JsonPropertyName("local_phone")]
        public string? LocalPhone { get; set; }

        [JsonPropertyName("mobile_phone")]
        public string? MobilePhone { get; set; }

        [JsonPropertyName("is_manager")]
        public string? IsManager { get; set; }

        [JsonPropertyName("manager_id")]
        public string? ManagerId { get; set; }

        [JsonPropertyName("disabled")]
        public string? Disabled { get; set; }

        [JsonPropertyName("tab_number")]
        public string? TabNumber { get; set; }
    }
}



