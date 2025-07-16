using System.DirectoryServices.Protocols;

namespace DinDin.Extensions;

public static class LdapExtensions
{
    public static string? GetString(this SearchResultEntry entry, string attributeName)
    {
        if (entry.Attributes.Contains(attributeName))
            return entry.Attributes[attributeName]?[0]?.ToString();

        return null;
    }

    public static string[]? GetStringArray(this SearchResultEntry entry, string attributeName)
    {
        if (entry.Attributes.Contains(attributeName))
        {
            return entry.Attributes[attributeName]?
                .Cast<object>()
                .Select(x => x.ToString())
                .Where(x => !string.IsNullOrWhiteSpace(x))
                .ToArray();
        }

        return null;
    }

    public static int GetInt(this SearchResultEntry entry, string attributeName)
    {
        var str = entry.GetString(attributeName);
        return int.TryParse(str, out int val) ? val : 0;
    }
}

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

        OnModelCreatingPartial(modelBuilder);
    }

    
    partial void OnModelCreatingPartial(ModelBuilder modelBuilder);
}

using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;
using System.DirectoryServices.Protocols;
using System.Reflection;
using System.Text.Json;
using System.Text.Json.Serialization;
using DinDin.Extensions;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace DinDin.Models;

public class LDAPEmployee
{
    [Key]
    [Column("id")]
    public int Id { get; set; }

    [Column("user_status_code")]
    public int UserStatusCode { get; set; }

    [Column("is_disabled")]
    public bool IsDisabled { get; set; }

    [Column("given_name")]
    public string? GivenName { get; set; }

    [Column("name")]
    public string Name { get; set; } = "";

    [Column("position_name")]
    public string? PositionName { get; set; }

    [Column("title")]
    public string? Title { get; set; }

    [Column("department")]
    public string? Department { get; set; }

    [Column("login")]
    public string Login { get; set; } = "";

    [Column("local_phone")]
    public string? LocalPhone { get; set; }

    [Column("mobile_number")]
    public string? MobileNumber { get; set; }

    [Column("city")]
    public string? City { get; set; }

    [Column("location_address")]
    public string? LocationAddress { get; set; }

    [Column("postal_code")]
    public string? PostalCode { get; set; }

    [Column("company")]
    public string? Company { get; set; }

    [Column("manager")]
    public string? Manager { get; set; }

    [Column("email")]
    public string? Email { get; set; }

    [Column("member_of")]
    public string[]? MemberOf { get; set; }

    [Column("member_of_string")]
    public string? MemberOfString { get; set; }

    [Column("department_id")]
    public int? DepartmentId { get; set; }

    [Column("department_details")]
    public JsonDocument? DepartmentDetails { get; set; }

    [Column("group_id")]
    public int? GroupId { get; set; }

    [Column("is_manager")]
    public bool IsManager { get; set; }

    public LDAPEmployee() {}

    public LDAPEmployee(SearchResultEntry entry)
    {
        UserStatusCode = entry.GetInt("userAccountControl");
        IsDisabled = (UserStatusCode & 2) != 0;
        GivenName = entry.GetString("givenName");
        Name = entry.GetString("cn") ?? "";
        PositionName = entry.GetString("position");
        Title = entry.GetString("title");
        Department = entry.GetString("department");
        Login = entry.GetString("sAMAccountName") ?? "";
        LocalPhone = entry.GetString("telephoneNumber");
        MobileNumber = entry.GetString("mobile");
        City = entry.GetString("l");
        LocationAddress = entry.GetString("streetAddress");
        PostalCode = entry.GetString("postalCode");
        Company = entry.GetString("company");
        Manager = entry.GetString("manager");
        Email = entry.GetString("mail");
        MemberOf = entry.GetStringArray("memberOf");
        MemberOfString = MemberOf == null ? null : string.Join(";", MemberOf);
        IsManager = false;
    }
}

using DinDin.Models;
using EFCore.BulkExtensions;
using Microsoft.EntityFrameworkCore;

namespace DinDin.Repositories;

public class LDAPUsersRepository
{
    private readonly BpmcoreContext _context;

    public LDAPUsersRepository(BpmcoreContext context)
    {
        _context = context;
    }

    public async Task UpdateByKey(List<LDAPEmployee> employees)
    {
        var logins = employees.Select(e => e.Login).ToList();
        var existing = await _context.LdapEmployees
            .Where(x => logins.Contains(x.Login))
            .ToListAsync();

        var toUpdate = employees.Where(e => existing.Any(x => x.Login == e.Login)).ToList();
        var toInsert = employees.Where(e => existing.All(x => x.Login != e.Login)).ToList();

        foreach (var updated in toUpdate)
        {
            var record = existing.First(x => x.Login == updated.Login);
            _context.Entry(record).CurrentValues.SetValues(updated);
        }

        _context.LdapEmployees.AddRange(toInsert);
        await _context.SaveChangesAsync();
    }
}

using System.DirectoryServices.Protocols;
using System.Net;
using DinDin.Models;
using Microsoft.Extensions.Configuration;

namespace DinDin.Services;

public class LdapEmployeeSyncService
{
    private readonly string _server;
    private readonly int _port;
    private readonly string _bindDn;
    private readonly string _password;
    private readonly string _searchBase;
    private readonly string _userFilter;

    public LdapEmployeeSyncService(IConfiguration configuration)
    {
        _server = configuration["LDAPConfig:Server"]!;
        _port = int.Parse(configuration["LDAPConfig:Port"]!);
        _bindDn = configuration["LDAPConfig:BindDN"]!;
        _password = configuration["LDAPConfig:Password"]!;
        _searchBase = configuration["LDAPConfig:SearchBase"]!;
        _userFilter = configuration["LDAPConfig:Filters:UserPerson:Code"]!;
    }

    public List<SearchResultEntry> GetLdapEntries()
    {
        var result = new List<SearchResultEntry>();

        using var ldapConnection = new LdapConnection(new LdapDirectoryIdentifier(_server, _port));
        ldapConnection.AuthType = AuthType.Basic;
        ldapConnection.Bind(new NetworkCredential(_bindDn, _password));

        var request = new SearchRequest(_searchBase, _userFilter, SearchScope.Subtree, null);

        var pageRequest = new PageResultRequestControl(500);
        request.Controls.Add(pageRequest);

        while (true)
        {
            var response = (SearchResponse)ldapConnection.SendRequest(request);
            result.AddRange(response.Entries.Cast<SearchResultEntry>());

            var pageResponse = response.Controls.OfType<PageResultResponseControl>().FirstOrDefault();
            if (pageResponse == null || pageResponse.Cookie.Length == 0)
                break;

            pageRequest.Cookie = pageResponse.Cookie;
        }

        return result;
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
        
        await SyncLdap(serviceProvider);
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
    
    static async Task SyncLdap(IServiceProvider provider)
    {
        var context = provider.GetRequiredService<BpmcoreContext>();
        var repo = new LDAPUsersRepository(context);
        var service = new LdapEmployeeSyncService();
        var employees = service.GetLdapEmployees();

        await repo.UpdateByKey(employees);
        Console.WriteLine($"✅ Импортировано: {employees.Count} записей в таблицу ldap_employees");
    }
}

