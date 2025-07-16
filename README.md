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
