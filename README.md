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
using System.Net;
using DinDin.Models;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;

namespace DinDin.Services;

public class LdapEmployeeSyncService
{
    private readonly ILogger<LdapEmployeeSyncService> _logger;
    private readonly string _server;
    private readonly int _port;
    private readonly string _bindDn;
    private readonly string _password;
    private readonly string _searchBase;
    private readonly string _filter;
    private readonly string[] _attributes;

    public LdapEmployeeSyncService(IConfiguration configuration, ILogger<LdapEmployeeSyncService> logger)
    {
        _logger = logger;

        _server = configuration["LDAPConfig:Server"] ?? throw new ArgumentNullException("Server");
        _port = int.TryParse(configuration["LDAPConfig:Port"], out var port) ? port : 389;
        _bindDn = configuration["LDAPConfig:BindDN"] ?? throw new ArgumentNullException("BindDN");
        _password = configuration["LDAPConfig:Password"] ?? throw new ArgumentNullException("Password");
        _searchBase = configuration["LDAPConfig:SearchBase"] ?? throw new ArgumentNullException("SearchBase");

        _filter = configuration["LDAPConfig:Filters:UserPerson:Code"] ?? "(objectClass=user)";
        _attributes = configuration.GetSection("LDAPConfig:Filters:UserPerson:Keys").Get<string[]>() ?? Array.Empty<string>();

        _logger.LogInformation("LDAP Config loaded: Server={Server}, Port={Port}, BindDN={BindDN}, SearchBase={SearchBase}", _server, _port, _bindDn, _searchBase);
    }

    public List<LDAPEmployee> GetLdapEmployees()
{
    _logger.LogInformation("Connecting to LDAP server {Server}:{Port}...", _server, _port);

    var credentials = new NetworkCredential(_bindDn, _password);
    var identifier = new LdapDirectoryIdentifier(_server, _port);

    using var connection = new LdapConnection(identifier, credentials, AuthType.Negotiate);
    connection.SessionOptions.ProtocolVersion = 3;

    try
    {
        connection.Bind();
        _logger.LogInformation("LDAP bind successful");
    }
    catch (LdapException ex)
    {
        _logger.LogError(ex, "LDAP bind failed: {Message}", ex.Message);
        throw;
    }

    var employees = new List<LDAPEmployee>();
    var pageSize = 1000;
    var cookie = Array.Empty<byte>();

    do
    {
        // Создаём новый запрос на каждой итерации
        var searchRequest = new SearchRequest(_searchBase, _filter, SearchScope.Subtree, _attributes);

        // Добавляем постраничную пагинацию
        var pageControl = new PageResultRequestControl(pageSize) { Cookie = cookie };
        searchRequest.Controls.Add(pageControl);

        // Выполняем запрос
        var response = (SearchResponse)connection.SendRequest(searchRequest);

        // Обрабатываем записи
        foreach (SearchResultEntry entry in response.Entries)
        {
            try
            {
                var emp = new LDAPEmployee(entry);
                employees.Add(emp);
            }
            catch (Exception ex)
            {
                _logger.LogWarning(ex, "Failed to parse entry");
            }
        }

        // Извлекаем cookie для следующей страницы
        var pageResponse = response.Controls.OfType<PageResultResponseControl>().FirstOrDefault();
        cookie = pageResponse?.Cookie ?? Array.Empty<byte>();

        _logger.LogInformation("Загружено ещё {Count} записей, всего: {Total}", response.Entries.Count, employees.Count);

    } while (cookie != null && cookie.Length > 0);

    _logger.LogInformation("✅ Всего импортировано из LDAP: {Count} записей", employees.Count);
    return employees;
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

    /// <summary>
    /// Полностью заменяет таблицу LdapEmployees — сначала очищает, затем вставляет все записи.
    /// </summary>
    public async Task ReplaceAllLdapEmployees(List<LDAPEmployee> employees)
    {
        // Очищаем таблицу
        await _context.Database.ExecuteSqlRawAsync("TRUNCATE TABLE \"ldap_employees\" RESTART IDENTITY");

        // Вставляем по батчам (например, по 1000)
        var bulkConfig = new BulkConfig
        {
            BatchSize = 1000,         // Можно увеличить до 2000–5000
            UseTempDB = true          // Улучшает производительность
        };

        await _context.BulkInsertAsync(employees, bulkConfig);
    }

    /// <summary>
    /// Обновляет поле LoginAd в таблице Employees, если mail совпадает с Email в LdapEmployees
    /// </summary>
    public async Task SyncEmployeeLoginsByEmail()
    {
        var ldapList = await _context.LdapEmployees
            .Where(x => !string.IsNullOrEmpty(x.Email))
            .ToListAsync();

        var emailToLogin = ldapList
            .GroupBy(e => e.Email.ToLower())
            .ToDictionary(g => g.Key, g => g.First().Login);

        var employees = await _context.Employees
            .Where(e => !string.IsNullOrEmpty(e.Mail))
            .ToListAsync();

        foreach (var emp in employees)
        {
            var email = emp.Mail?.ToLower();
            if (email != null && emailToLogin.TryGetValue(email, out var login))
            {
                emp.LoginAd = login;
            }
        }

        await _context.SaveChangesAsync();
    }
}


{
  "ConnectionStrings": {
    "BpmCore": "Host=localhost;Port=5432;Username=postgres;Password=postgres;Database=bpmbase_local;SslMode=Disable;SearchPath=default"
  },

  "LDAPConfig": {
    "Server": "172.31.0.252",
    "Port": "389",
    "BindDN": "gnpfadm@enpf.kz",
    "Password": "EghfdktybtFLV$77#",
    "SearchBase": "dc=enpf,dc=kz",
    "Filters": {
      "UserPerson": {
        "Code": "(&(objectClass=user)(objectCategory=PERSON))",
        "Keys": [
          "givenName",
          "sAMAccountName",
          "department",
          "member",
          "memberof",
          "manager",
          "cn",
          "UserAccountControl",
          "mail",
          "sn",
          "wWWWHomePage",
          "homePhone",
          "streetAddress",
          "postOfficeBox",
          "l",
          "st",
          "postalCode",
          "c",
          "company",
          "title",
          "telephoneNumber",
          "facsimileTelephoneNumber",
          "description",
          "ipPhone"
        ]
      }
    }
  }

}
