using DinDin.Models;
using Microsoft.Extensions.Configuration;
using System.DirectoryServices.Protocols;

namespace DinDin.Services.Ldap;

internal class UserLdapService : LDAPService
{
    public UserLdapService(IConfiguration configuration) : base(configuration) { }

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


using DinDin.Models;
using DinDin.Persistence;
using EFCore.BulkExtensions;

namespace DinDin.Repositories;

internal class LDAPUsersRepository(BpmCoreContext context)
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



private static async Task SyncLdapAsync(IServiceProvider serviceProvider)
{
    var configuration = serviceProvider.GetRequiredService<IConfiguration>();
    var context = serviceProvider.GetRequiredService<BpmCoreContext>();

    var ldapService = new UserLdapService(configuration);
    var employees = ldapService.GetLDAPEmployees();

    var repository = new LDAPUsersRepository(context);
    await repository.UpdateByKey(employees);

    Console.WriteLine($"✅ LDAP синхронизация завершена. Обновлено: {employees.Count} записей.");
}


using DinDin.Models;
using Microsoft.EntityFrameworkCore;

namespace DinDin.Persistence
{
    public class BpmCoreContext : DbContext
    {
        public BpmCoreContext(DbContextOptions<BpmCoreContext> options)
            : base(options)
        {
        }

        public DbSet<LDAPEmployee> LdapEmployees => Set<LDAPEmployee>();

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);

            // Конфигурация из самого LDAPEmployee
            modelBuilder.ApplyConfiguration(new LDAPEmployee());
        }
    }
}

