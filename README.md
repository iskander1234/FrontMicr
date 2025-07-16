using System.Text.Json.Serialization;
using System.DirectoryServices.Protocols;
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;
using Microsoft.EntityFrameworkCore;

namespace DinDin.Models;

[Table("ldap_employees")]
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
    public string Name { get; set; } = string.Empty;

    [Column("position_name")]
    public string? PositionName { get; set; }

    [Column("title")]
    public string? Title { get; set; }

    [Column("department")]
    public string? Department { get; set; }

    [Column("login")]
    public string Login { get; set; } = string.Empty;

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

    [Column("department_details", TypeName = "jsonb")]
    public string? DepartmentDetails { get; set; }

    [Column("group_id")]
    public int? GroupId { get; set; }

    [Column("is_manager")]
    public bool IsManager { get; set; } = false;

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
        DepartmentDetails = null; // Заполни если нужно
    }
}



public static class LdapExtensions
{
    public static string? GetString(this SearchResultEntry entry, string attrName)
    {
        return entry.Attributes[attrName]?.Count > 0 ? entry.Attributes[attrName][0]?.ToString() : null;
    }

    public static string[]? GetStringArray(this SearchResultEntry entry, string attrName)
    {
        return entry.Attributes[attrName]?.Cast<string>().ToArray();
    }

    public static int GetInt(this SearchResultEntry entry, string attrName)
    {
        return int.TryParse(entry.GetString(attrName), out var val) ? val : 0;
    }
}



public virtual DbSet<LDAPEmployee> LdapEmployees => Set<LDAPEmployee>();


using DinDin.Models;
using Microsoft.EntityFrameworkCore;

namespace DinDin.Repositories;

public class LDAPUsersRepository
{
    private readonly BpmcoreContext _context;

    public LDAPUsersRepository(BpmcoreContext context)
    {
        _context = context;
    }

    public async Task BulkUpsertAsync(List<LDAPEmployee> employees)
    {
        // Удалим всё и вставим заново
        _context.LdapEmployees.RemoveRange(_context.LdapEmployees);
        await _context.SaveChangesAsync();

        await _context.LdapEmployees.AddRangeAsync(employees);
        await _context.SaveChangesAsync();
    }
}






public static class LdapSyncRunner
{
    public static async Task SyncAsync(IServiceProvider serviceProvider)
    {
        var config = serviceProvider.GetRequiredService<IConfiguration>();
        var context = serviceProvider.GetRequiredService<BpmcoreContext>();

        var ldapService = new LDAPServiceImpl(config); // см. ниже
        var entries = ldapService.LdapPagedSearch();

        var employees = entries.Select(e => new LDAPEmployee(e)).ToList();

        var repository = new LDAPUsersRepository(context);
        await repository.BulkUpsertAsync(employees);

        Console.WriteLine($"✅ Загружено {employees.Count} записей из LDAP.");
    }
}


namespace DinDin.Services;

public class LDAPServiceImpl : LDAPService
{
    public LDAPServiceImpl(IConfiguration config) : base(config) { }
}




public class Program
{
    public static async Task Main(string[] args)
    {
        var config = new ConfigurationBuilder()
            .AddJsonFile("appsettings.json")
            .Build();

        var services = new ServiceCollection();
        services.AddSingleton<IConfiguration>(config);
        services.AddDbContext<BpmcoreContext>();
        services.AddScoped<LDAPUsersRepository>();

        var serviceProvider = services.BuildServiceProvider();

        await LdapSyncRunner.SyncAsync(serviceProvider);
    }
}
