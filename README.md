using System.DirectoryServices.Protocols;
using System.Text.Json;
using System.Text.Json.Serialization;
using System.ComponentModel.DataAnnotations.Schema;
using System.ComponentModel.DataAnnotations;
using DinDin.Extensions;

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

namespace DinDin.Services;

public class LdapEmployeeSyncService
{
    private readonly string _host = "ldap://yourhost";
    private readonly int _port = 389;
    private readonly string _username = "cn=admin,dc=your,dc=org";
    private readonly string _password = "yourpassword";
    private readonly string _baseDn = "dc=your,dc=org";

    public List<LDAPEmployee> GetLdapEmployees()
    {
        var result = new List<LDAPEmployee>();

        using var connection = new LdapConnection(new LdapDirectoryIdentifier(_host, _port));
        connection.AuthType = AuthType.Basic;
        connection.Bind(new NetworkCredential(_username, _password));

        var pageRequest = new PageResultRequestControl(1000);
        var searchRequest = new SearchRequest(_baseDn, "(objectClass=person)", SearchScope.Subtree);
        searchRequest.Controls.Add(pageRequest);

        while (true)
        {
            var response = (SearchResponse)connection.SendRequest(searchRequest);
            foreach (SearchResultEntry entry in response.Entries)
            {
                result.Add(new LDAPEmployee(entry));
            }

            var prc = response.Controls.OfType<PageResultResponseControl>().FirstOrDefault();
            if (prc == null || prc.Cookie.Length == 0) break;

            pageRequest.Cookie = prc.Cookie;
        }

        return result;
    }
}


using DinDin.Repositories;
using DinDin.Services;

static async Task SyncLdap(IServiceProvider provider)
{
    var context = provider.GetRequiredService<BpmcoreContext>();
    var repo = new LDAPUsersRepository(context);
    var service = new LdapEmployeeSyncService();
    var employees = service.GetLdapEmployees();

    await repo.UpdateByKey(employees);
    Console.WriteLine($"✅ Импортировано: {employees.Count} записей в таблицу ldap_employees");
}


await SyncLdap(serviceProvider);
