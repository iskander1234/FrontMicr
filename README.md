"C:\Program Files\JetBrains\JetBrains Rider 2025.1.2\plugins\dpa\DotFiles\JetBrains.DPA.Runner.exe" --handle=21288 --backend-pid=8868 --etw-collect-flags=67108622 --detach-event-name=dpa.detach.8868.25 --refresh-interval=1 -- C:/BPM/Leshan/1/DinDin/bin/Debug/net8.0/DinDin.exe
Unhandled exception. System.InvalidOperationException: No service for type 'Microsoft.Extensions.Configuration.IConfiguration' has been registered.
   at Microsoft.Extensions.DependencyInjection.ServiceProviderServiceExtensions.GetRequiredService(IServiceProvider provider, Type serviceType)
   at Microsoft.Extensions.DependencyInjection.ServiceProviderServiceExtensions.GetRequiredService[T](IServiceProvider provider)
   at DinDin.Program.SyncLdapAsync(IServiceProvider serviceProvider) in C:\BPM\Leshan\1\DinDin\Program.cs:line 125
   at DinDin.Program.StartData(ServiceProvider serviceProvider) in C:\BPM\Leshan\1\DinDin\Program.cs:line 75
   at DinDin.Program.Main() in C:\BPM\Leshan\1\DinDin\Program.cs:line 24
   at DinDin.Program.<Main>()

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
        
        // Конфигурация из самого LDAPEmployee
        modelBuilder.ApplyConfiguration(new LDAPEmployee());

        OnModelCreatingPartial(modelBuilder);
    }

    
    partial void OnModelCreatingPartial(ModelBuilder modelBuilder);
}

using System.ComponentModel.DataAnnotations.Schema;
using System.DirectoryServices.Protocols;
using System.Reflection;
using System.Text.Json.Serialization;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace DinDin.Models;

[Serializable]
public record LDAPEmployee : IEntityTypeConfiguration<LDAPEmployee>
{
    [Column(name: "id")]
    public int Id { get; set; }

    [Column(name: "user_status_code")]
    [JsonPropertyName("UserAccountControl")]
    public int UserStatusCode { get; set; }

    [Column(name: "is_disabled")]
    public bool IsDisabled { get; set; }

    [Column(name: "given_name", TypeName = "varchar")]
    [JsonPropertyName("givenName")]
    public string? GivenName { get; set; } = null!;

    [Column(name: "name", TypeName = "varchar")]
    [JsonPropertyName("cn")]
    public string Name { get; set; } = null!;

    [Column(name: "position_name", TypeName = "varchar")]
    [JsonPropertyName("description")]
    public string? PositionName { get; set; } = null!;

    [Column(name: "title", TypeName = "varchar")]
    [JsonPropertyName("title")]
    public string? Title { get; set; } = null!;

    [Column(name: "department", TypeName = "varchar")]
    [JsonPropertyName("department")]
    public string? Department { get; set; } = null!;

    [Column(name: "login", TypeName = "varchar")]
    [JsonPropertyName("sAMAccountName")]
    public string Login { get; set; } = null!;

    [Column(name: "local_phone", TypeName = "varchar")]
    [JsonPropertyName("ipPhone")]
    public string? LocalPhone { get; set; } = null!;

    [Column(name: "mobile_number", TypeName = "varchar")]
    [JsonPropertyName("telephoneNumber")]
    public string? MobileNumber { get; set; } = null!;

    [Column(name: "city", TypeName = "varchar")]
    [JsonPropertyName("l")]
    public string? City { get; set; } = null!;

    [Column(name: "location_address", TypeName = "varchar")]
    [JsonPropertyName("streetAddress")]
    public string? LocationAddress { get; set; } = null!;

    [Column(name: "postal_code", TypeName = "varchar")]
    [JsonPropertyName("postalCode")]
    public string? PostalCode { get; set; } = null!;

    [Column(name: "company", TypeName = "varchar")]
    [JsonPropertyName("company")]
    public string? Company { get; set; } = null!;

    [Column(name: "manager", TypeName = "varchar")]
    [JsonPropertyName("manager")]
    public string? Manager { get; set; } = null!;

    [Column(name: "email", TypeName = "varchar")]
    [JsonPropertyName("mail")]
    public string? Email { get; set; } = null!;

    [Column(name: "member_of")]
    [JsonPropertyName("memberof")]
    public List<string>? MemberOf { get; set; } = null!;

    [Column(name: "member_of_string", TypeName = "varchar")]
    public string? MemberOfString { get; set; } = null!;

    [Column(name: "is_manager")]
    public bool IsManager { get; set; } = false;

    [Column(name: "group_id")]
    public int? GroupId { get; set; } = null!;

    [Column(name: "department_id")]
    public int? DepartmentId { get; set; } = null!;

    [Column(name: "department_details", TypeName = "jsonb")]
    public List<DepartmentDetail>? DepartmentDetails { get; set; } = null!;

    public LDAPEmployee() { }

    public LDAPEmployee(in SearchResultEntry entry)
    {
        var properties = typeof(LDAPEmployee).GetProperties();

        foreach (var property in properties)
        {
            var jsonName = property.GetCustomAttribute<JsonPropertyNameAttribute>()?.Name;

            if (jsonName == null || !entry.Attributes.Contains(jsonName))
                continue;

            var value = entry.Attributes[jsonName][0]?.ToString();
            if (string.IsNullOrEmpty(value))
                return;

            SetByJsonName(property, jsonName, value);
        }
    }

    private string? SetByJsonName(in PropertyInfo prop, string? jsonName, string value)
    {
        if (jsonName == "memberof")
            SetMemberOf(value);
        else if (jsonName == "manager" && value.Contains(','))
            value = value.Split(',')[0].Replace("CN=", "").Trim();
        else if (prop.PropertyType == typeof(int) && int.TryParse(value, out var intValue))
        {
            if (jsonName == "UserAccountControl")
                IsDisabled = IsAccountDisabled(intValue);
            prop.SetValue(this, intValue);
        }
        else prop.SetValue(this, value);

        return value;
    }

    private void SetMemberOf(string value)
    {
        MemberOf ??= [];
        MemberOfString = value.Replace(",DC=enpf,DC=kz", "");
        MemberOf = MemberOfString!
            .Replace(",OU=Группы", "")
            .Replace(",CN=Users", "")
            .Replace("OU=", "")
            .Replace("CN=", "")
            .Split(",")
            .Distinct()
            .ToList();
    }

    static bool IsAccountDisabled(int userAccountControlValue)
    {
        const int ACCOUNTDISABLE = 0x0002;
        return (userAccountControlValue & ACCOUNTDISABLE) == ACCOUNTDISABLE;
    }

    public void Configure(EntityTypeBuilder<LDAPEmployee> builder)
    {
        builder.ToTable("ldap_employees", "public", e => e.ExcludeFromMigrations());
        builder.HasKey(e => e.Id);
        builder.Property(e => e.Id).UseIdentityColumn();
        builder.HasIndex(e => e.Id);
        builder.Property(e => e.PositionName).IsRequired(false);
        builder.HasIndex(e => e.Login).IsUnique();
    }
}

using System.Text.Json.Serialization;

namespace DinDin.Models;

[JsonConverter(typeof(JsonStringEnumConverter))]
public class DepartmentDetail
{
    public int Id { get; set; }
    public string LongName { get; set; } = null!;
}


using DinDin.Models;
using EFCore.BulkExtensions;

namespace DinDin.Repositories;

internal class LDAPUsersRepository(BpmcoreContext context)
{
    public async Task UpdateByKey(List<LDAPEmployee> employees)
    {
        await context.BulkInsertOrUpdateAsync(employees.ToList(), options =>
        {
            options.BatchSize = 1000;
            options.IncludeGraph = false;
            options.UpdateByProperties = ["Login"];
            options.SetOutputIdentity = true;
            options.UseTempDB = false;
            options.PropertiesToExclude =
            [
                "Id", "GroupId", "DepartmentId", "IsManager"
            ];
        });
    }
}


using System.DirectoryServices.Protocols;
using System.Net;
using Microsoft.Extensions.Configuration;

namespace DinDin.Services;

internal abstract class LDAPService
{
    protected readonly string LdapHost;
    protected readonly int LdapPort;
    protected readonly string LdapUser;
    protected readonly string LdapPassword;
    protected readonly string LdapSearchBase;

    protected virtual string Filter => "(objectClass=person)";

    protected LDAPService(IConfiguration configuration)
    {
        LdapHost = configuration["Ldap:Host"]!;
        LdapPort = int.Parse(configuration["Ldap:Port"]!);
        LdapUser = configuration["Ldap:User"]!;
        LdapPassword = configuration["Ldap:Password"]!;
        LdapSearchBase = configuration["Ldap:SearchBase"]!;
    }

    protected List<SearchResultEntry> LdapPagedSearch()
    {
        var results = new List<SearchResultEntry>();
        using var ldapConnection = new LdapConnection(new LdapDirectoryIdentifier(LdapHost, LdapPort));
        ldapConnection.AuthType = AuthType.Basic;
        ldapConnection.Bind(new NetworkCredential(LdapUser, LdapPassword));

        var pageRequestControl = new PageResultRequestControl(1000);
        var searchRequest = new SearchRequest(LdapSearchBase, Filter, SearchScope.Subtree, null);
        searchRequest.Controls.Add(pageRequestControl);

        while (true)
        {
            var searchResponse = (SearchResponse)ldapConnection.SendRequest(searchRequest);
            results.AddRange(searchResponse.Entries.Cast<SearchResultEntry>());

            var pageResponse = searchResponse.Controls.OfType<PageResultResponseControl>().FirstOrDefault();
            if (pageResponse == null || pageResponse.Cookie.Length == 0) break;

            pageRequestControl.Cookie = pageResponse.Cookie;
        }

        return results;
    }
}

using DinDin.Models;
using Microsoft.Extensions.Configuration;

namespace DinDin.Services;

internal class UserLdapService : LDAPService { public UserLdapService(IConfiguration configuration) : base(configuration) { }

    public List<LDAPEmployee> GetLDAPEmployees()
    {
        var entries = LdapPagedSearch();

        var employees = new List<LDAPEmployee>();
        foreach (var entry in entries)
        {
            var employee = new LDAPEmployee
            {
                GivenName = entry.Attributes["givenName"]?[0]?.ToString(),
                Name = entry.Attributes["cn"]?[0]?.ToString(),
                Login = entry.Attributes["sAMAccountName"]?[0]?.ToString(),
                Email = entry.Attributes["mail"]?[0]?.ToString(),
                UserStatusCode = int.TryParse(entry.Attributes["userAccountControl"]?[0]?.ToString(), out var code) ? code : 0,
                IsDisabled = false
            };

            employees.Add(employee);
        }

        return employees;
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
        await SyncLdapAsync(serviceProvider);
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
    
    private static async Task SyncLdapAsync(IServiceProvider serviceProvider)
    {
        var configuration = serviceProvider.GetRequiredService<IConfiguration>();
        var context = serviceProvider.GetRequiredService<BpmcoreContext>();

        var ldapService = new UserLdapService(configuration);
        var employees = ldapService.GetLDAPEmployees();

        var repository = new LDAPUsersRepository(context);
        await repository.UpdateByKey(employees);

        Console.WriteLine($"✅ LDAP синхронизация завершена. Обновлено: {employees.Count} записей.");
    }

}


{
  "ConnectionStrings": {
    "BpmCore": "Host=localhost;Port=5432;Username=postgres;Password=postgres;Database=bpmbase7;SslMode=Disable;SearchPath=default"
  },

  "Ldap": {
    "Host": "1123123",
    "Port": 389,
    "User": "qwerwer",
    "Password": "Password",
    "SearchBase": "SearchBase"
  }
}
