using DinDin.Models;
using EFCore.BulkExtensions;
using Microsoft.EntityFrameworkCore;

namespace DinDin.Repositories
{
    public class LDAPUsersRepository
    {
        private readonly BpmcoreContext _ctx;
        public LDAPUsersRepository(BpmcoreContext ctx) => _ctx = ctx;

        /// <summary>
        /// Полностью заменяет таблицу ldap_employees:
        /// TRUNCATE + BulkInsert.
        /// </summary>
        public async Task UpdateByKey(List<LDAPEmployee> updatedLdapEmployees)
        {
            await _ctx.BulkInsertOrUpdateAsync(updatedLdapEmployees.ToList(), options =>
            {
                options.BatchSize = 1000;
                options.IncludeGraph = false;
                options.UpdateByProperties = ["Login"];
                options.SetOutputIdentity = true;
                options.UseTempDB = false;
                options.PropertiesToExclude =
                [
                    "Id", "GroupId", "DepartmentId",
                    //"DepartmentDetails",
                    "IsManager"
                ];
            });
        }
        /// <summary>
        /// Обновляет LoginAd в основной таблице Employees по совпадению email.
        /// </summary>
        public async Task SyncEmployeeLoginsByEmail()
        {
            var ldapList = await _ctx.LdapEmployees
                .Where(x => !string.IsNullOrEmpty(x.Email))
                .ToListAsync();

            var map = ldapList
                .GroupBy(e => e.Email!.ToLower())
                .ToDictionary(g => g.Key, g => g.First().Login);

            var employees = await _ctx.Employees
                .Where(e => !string.IsNullOrEmpty(e.Mail))
                .ToListAsync();

            foreach (var emp in employees)
            {
                var mail = emp.Mail!.ToLower();
                if (map.TryGetValue(mail, out var login))
                    emp.LoginAd = login;
            }

            await _ctx.SaveChangesAsync();
        }
    }
}

using System.DirectoryServices.Protocols;
using System.Net;
using Microsoft.Extensions.Configuration;

namespace DinDin.Services;

internal abstract class LDAPService
{
    private const int PageSize = 1000;
    private readonly string _ldapServer;
    private readonly int _ldapPort;
    private readonly string _ldapBindDN;
    private readonly string _ldapPassword;
    private readonly string _ldapSearchBase;
    private readonly string _ldapCurrentFilter;
    private readonly string[] _filterKeys;

    public virtual string Filter => "UserPerson";

    protected LDAPService(IConfiguration configuration)
    {
        var ldapSettings = configuration.GetSection("LDAPConfig");
        _ldapServer = ldapSettings["Server"] ?? throw new NullReferenceException("LDAP Server is not specified");
        _ldapPort = int.Parse(ldapSettings["Port"]);
        _ldapBindDN = ldapSettings["BindDN"] ?? throw new NullReferenceException("BindDN is not specified");
        _ldapPassword = ldapSettings["Password"] ?? throw new NullReferenceException("Password is not specified");
        _ldapSearchBase = ldapSettings["SearchBase"] ?? throw new NullReferenceException("SearchBase is not specified");

        _ldapCurrentFilter = ldapSettings.GetSection($"Filters:{Filter}:Code").Value ?? throw new NullReferenceException(Filter + ":Code filter is not specified");

        _filterKeys = ldapSettings.GetSection($"Filters:{Filter}:Keys").GetChildren()?.Select(e => e.Value)?.ToArray() ?? throw new NullReferenceException(Filter + " Keys filter is not specified"); ;
    }

    public IEnumerable<SearchResultEntry> LdapPagedSearch()
    {
        using var ldapConnection = new LdapConnection(new LdapDirectoryIdentifier(_ldapServer + ":" + _ldapPort));
        ldapConnection.Credential = new NetworkCredential(_ldapBindDN, _ldapPassword);
        ldapConnection.SessionOptions.ProtocolVersion = 3;
        ldapConnection.Bind();

        var searchRequest = GetRequest(_ldapSearchBase, _ldapCurrentFilter, _filterKeys, SearchScope.Subtree);
        var pageRequest = new PageResultRequestControl(PageSize);
        searchRequest.Controls.Add(pageRequest);

        while (true)
        {
            SearchResponse searchResponse;
            try
            {
                searchResponse = (SearchResponse)ldapConnection.SendRequest(searchRequest);
            }
            catch (Exception e)
            {
                Console.WriteLine(_ldapCurrentFilter);
                Console.WriteLine("\nUnexpected exception occurred:\n\t{0}: {1}", e.GetType().Name, e.Message);
                yield break;
            }

            if (searchResponse.Controls.Length != 1 || !(searchResponse.Controls[0] is PageResultResponseControl pageResponse))
            {
                Console.WriteLine("Server does not support paging");
                yield break;
            }

            foreach (SearchResultEntry entry in searchResponse.Entries)
            {
                yield return entry;
            }

            if (pageResponse.Cookie.Length == 0)
                break;

            pageRequest.Cookie = pageResponse.Cookie;
        }
    }


    private static SearchRequest GetRequest(string dn, string filter, string[] returnAttrs, SearchScope scope = SearchScope.Subtree)
    {
        var request = new SearchRequest(dn, filter, scope, returnAttrs);

        // Set controls for security descriptor and domain scope
        var searchControl = new SearchOptionsControl(System.DirectoryServices.Protocols.SearchOption.DomainScope);
        var securityDescriptorFlagControl = new SecurityDescriptorFlagControl
        {
            SecurityMasks = SecurityMasks.Dacl | SecurityMasks.Owner
        };
        request.Controls.Add(securityDescriptorFlagControl);
        request.Controls.Add(searchControl);

        return request;
    }

    public void PrintAttributes(in SearchResultEntry entry)
    {
        foreach (var key in _filterKeys)
        {
            var selectedEntry = entry.Attributes[key];
            if (selectedEntry != null)
            {
                var value = selectedEntry != null ? selectedEntry[0].ToString() : null;

                if (value != null)
                    Console.WriteLine($"{key}: {value}");
            }
        }
        Console.WriteLine();
    }
}
using DinDin.Interface;
using DinDin.Models;
using Microsoft.Extensions.Configuration;

namespace DinDin.Services;

internal class UserLDAPService(IConfiguration configuration) : LDAPService(configuration), IUserLdapService
{
    public override string Filter => "UserPerson";

    public List<LDAPEmployee> GetLDAPEmployees()
    {
        var results = LdapPagedSearch();
        var employeeResults = results
            .Select(e => new LDAPEmployee(e))
            .ToList();
        return employeeResults;
    }
}
using System.DirectoryServices.Protocols;
using DinDin.Models;

namespace DinDin.Interface;

internal interface IUserLdapService
{
    public void PrintAttributes(in SearchResultEntry entry);
    public List<LDAPEmployee> GetLDAPEmployees();
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
using DinDin.Interface;
using Microsoft.Extensions.Logging;

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
        Configuration = BuildConfiguration();
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

        await UpdateLdapEmployees(serviceProvider,Configuration);
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
            .AddLogging(builder =>
            {
                builder.AddConsole(); // Включаем логи в консоль
                builder.SetMinimumLevel(LogLevel.Information);
            })
            .AddSingleton<IConfiguration>(configuration)
            .AddDbContext<BpmcoreContext>(options => options.UseNpgsql(connectionString))
            .AddSingleton<OneCService>()
            .AddSingleton<DepartmentRepository>()
            .AddSingleton<EmployeeRepository>()
            .AddSingleton<IUserLdapService, UserLDAPService>()
            .AddSingleton<LDAPService>()   // 👈 Добавь эту строку
            .AddSingleton<LDAPUsersRepository>()       // 👈 И эту
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
    
    // private static async Task SyncLdap(IServiceProvider provider)
    // {
    //     var context = provider.GetRequiredService<BpmcoreContext>();
    //     var ldapService = provider.GetRequiredService<LdapEmployeeSyncService>();
    //     var ldapUsersRepo = provider.GetRequiredService<LDAPUsersRepository>();
    //
    //     // Получаем всех LDAP сотрудников (без ограничения 1000)
    //     var ldapEmployees = ldapService.GetLdapEmployees();
    //
    //     // Полная замена таблицы LdapEmployees
    //     await ldapUsersRepo.ReplaceAllLdapEmployees(ldapEmployees);
    //
    //     Console.WriteLine($" Импортировано {ldapEmployees.Count} записей в таблицу ldap_employees.");
    //
    //     // Обновляем LoginAd в таблице Employees по email
    //     await ldapUsersRepo.SyncEmployeeLoginsByEmail();
    //     Console.WriteLine($" Обновлены поля LoginAd в таблице employees.");
    // }
    
    // private static async Task SyncLdap(IServiceProvider provider)
    // {
    //     var ldapService   = provider.GetRequiredService<LdapEmployeeSyncService>();
    //     var ldapUsersRepo = provider.GetRequiredService<LDAPUsersRepository>();
    //
    //     var allEmployees = ldapService.GetLdapEmployees();
    //     Console.WriteLine($"🔄 Retrieved {allEmployees.Count} LDAP records");
    //
    //     await ldapUsersRepo.ReplaceAllLdapEmployees(allEmployees);
    //     Console.WriteLine($"✅ Imported {allEmployees.Count} into ldap_employees table");
    //
    //     await ldapUsersRepo.SyncEmployeeLoginsByEmail();
    //     Console.WriteLine("🔄 Updated LoginAd in Employees table");
    // }

    
    private static async Task UpdateLdapEmployees(ServiceProvider serviceProvider, IConfiguration configuration)
    {
        var ldapEmployees = new UserLDAPService(configuration).GetLDAPEmployees();

        await serviceProvider
            .GetRequiredService<LDAPUsersRepository>()
            .UpdateByKey(ldapEmployees);
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
using DinDin.Interface;
using Microsoft.Extensions.Logging;

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
        Configuration = BuildConfiguration();
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

        await UpdateLdapEmployees(serviceProvider,Configuration);
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
            .AddLogging(builder =>
            {
                builder.AddConsole(); // Включаем логи в консоль
                builder.SetMinimumLevel(LogLevel.Information);
            })
            .AddSingleton<IConfiguration>(configuration)
            .AddDbContext<BpmcoreContext>(options => options.UseNpgsql(connectionString))
            .AddSingleton<OneCService>()
            .AddSingleton<DepartmentRepository>()
            .AddSingleton<EmployeeRepository>()
            .AddSingleton<LDAPUsersRepository>()       // 👈 И эту
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
    
    // private static async Task SyncLdap(IServiceProvider provider)
    // {
    //     var context = provider.GetRequiredService<BpmcoreContext>();
    //     var ldapService = provider.GetRequiredService<LdapEmployeeSyncService>();
    //     var ldapUsersRepo = provider.GetRequiredService<LDAPUsersRepository>();
    //
    //     // Получаем всех LDAP сотрудников (без ограничения 1000)
    //     var ldapEmployees = ldapService.GetLdapEmployees();
    //
    //     // Полная замена таблицы LdapEmployees
    //     await ldapUsersRepo.ReplaceAllLdapEmployees(ldapEmployees);
    //
    //     Console.WriteLine($" Импортировано {ldapEmployees.Count} записей в таблицу ldap_employees.");
    //
    //     // Обновляем LoginAd в таблице Employees по email
    //     await ldapUsersRepo.SyncEmployeeLoginsByEmail();
    //     Console.WriteLine($" Обновлены поля LoginAd в таблице employees.");
    // }
    
    // private static async Task SyncLdap(IServiceProvider provider)
    // {
    //     var ldapService   = provider.GetRequiredService<LdapEmployeeSyncService>();
    //     var ldapUsersRepo = provider.GetRequiredService<LDAPUsersRepository>();
    //
    //     var allEmployees = ldapService.GetLdapEmployees();
    //     Console.WriteLine($"🔄 Retrieved {allEmployees.Count} LDAP records");
    //
    //     await ldapUsersRepo.ReplaceAllLdapEmployees(allEmployees);
    //     Console.WriteLine($"✅ Imported {allEmployees.Count} into ldap_employees table");
    //
    //     await ldapUsersRepo.SyncEmployeeLoginsByEmail();
    //     Console.WriteLine("🔄 Updated LoginAd in Employees table");
    // }

    
    private static async Task UpdateLdapEmployees(ServiceProvider serviceProvider, IConfiguration configuration)
    {
        var ldapEmployees = new UserLDAPService(configuration).GetLDAPEmployees();

        await serviceProvider
            .GetRequiredService<LDAPUsersRepository>()
            .UpdateByKey(ldapEmployees);
    } 
    
}


