**public List<LDAPEmployee> GetLdapEmployees()
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

    _logger.LogInformation("Sending paged LDAP search request with filter: {Filter}", _filter);

    var employees = new List<LDAPEmployee>();

    var pageSize = 1000;
    var cookie = Array.Empty<byte>();
    var searchRequest = new SearchRequest(_searchBase, _filter, SearchScope.Subtree, _attributes);

    do
    {
        var pageControl = new PageResultRequestControl(pageSize) { Cookie = cookie };
        searchRequest.Controls.Clear();
        searchRequest.Controls.Add(pageControl);

        var response = (SearchResponse)connection.SendRequest(searchRequest);

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

        var pageResponse = response.Controls
            .OfType<PageResultResponseControl>()
            .FirstOrDefault();

        cookie = pageResponse?.Cookie ?? Array.Empty<byte>();

    } while (cookie != null && cookie.Length > 0);

    _logger.LogInformation("✅ Импортировано {Count} записей из LDAP.", employees.Count);
    return employees;
}
**"C:\Program Files\JetBrains\JetBrains Rider 2025.1.2\plugins\dpa\DotFiles\JetBrains.DPA.Runner.exe" --handle=7856 --backend-pid=8868 --etw-collect-flags=67108622 --detach-event-name=dpa.detach.8868.72 --refresh-interval=1 -- C:/BPM/Leshan/1/DinDin/bin/Debug/net8.0/DinDin.exe
warn: Microsoft.EntityFrameworkCore.Model.Validation[10400]
      Sensitive data logging is enabled. Log entries and exception messages may include sensitive application data; this mode should only be enabled during development.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (182ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      CREATE TABLE "public"."DepartmentsTempe70c08e5" AS TABLE "public"."Departments" WITH NO DATA;
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (31ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO "public"."Departments" ("id", "actual", "manager_id", "name", "parent_id") (SELECT "id", "actual", "manager_id", "name", "parent_id" FROM "public"."DepartmentsTempe70c08e5") ON CONFLICT ("id") DO UPDATE SET "actual" = EXCLUDED."actual", "manager_id" = EXCLUDED."manager_id", "name" = EXCLUDED."name", "parent_id" = EXCLUDED."parent_id" RETURNING "public"."Departments"."id", "public"."Departments"."actual", "public"."Departments"."manager_id", "public"."Departments"."name", "public"."Departments"."parent_id"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (24ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      DROP TABLE IF EXISTS "public"."DepartmentsTempe70c08e5"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (11ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      CREATE TABLE "public"."DepartmentsTemp5fe21a78" AS TABLE "public"."Departments" WITH NO DATA;
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (22ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO "public"."Departments" ("id", "actual", "manager_id", "name", "parent_id") (SELECT "id", "actual", "manager_id", "n
ame", "parent_id" FROM "public"."DepartmentsTemp5fe21a78") ON CONFLICT ("id") DO UPDATE SET "actual" = EXCLUDED."actual", "manager_id
" = EXCLUDED."manager_id", "name" = EXCLUDED."name", "parent_id" = EXCLUDED."parent_id" RETURNING "public"."Departments"."id", "public"."Departments"."actual", "public"."Departments"."manager_id", "public"."Departments"."name", "public"."Departments"."parent_id"    
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (2ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      DROP TABLE IF EXISTS "public"."DepartmentsTemp5fe21a78"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (18ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT e.id, e.dep_id, e.dep_name, e.disabled, e.is_filial, e.is_manager, e.local_phone, e.login, e.login_ad, e.mail, e.manager
_tab_number, e.mobile_phone, e.name, e.parent_dep_id, e.parent_dep_name, e.position, e.status_code, e.status_description, e.tab_number
      FROM public."Employees" AS e
info: DinDin.Services.LdapEmployeeSyncService[0]
      LDAP Config loaded: Server=172.31.0.252, Port=389, BindDN=gnpfadm@enpf.kz, SearchBase=dc=enpf,dc=kz
info: DinDin.Services.LdapEmployeeSyncService[0]
      Connecting to LDAP server 172.31.0.252:389...
info: DinDin.Services.LdapEmployeeSyncService[0]
      LDAP bind successful
info: DinDin.Services.LdapEmployeeSyncService[0]
      Sending paged LDAP search request with filter: (&(objectClass=user)(objectCategory=PERSON))
info: DinDin.Services.LdapEmployeeSyncService[0]
      ? Импортировано 1000 записей из LDAP.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (43ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      TRUNCATE TABLE "ldap_employees" RESTART IDENTITY
 Импортировано 1000 записей в таблицу ldap_employees.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (3ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT l.id, l.city, l.company, l.department, l.department_details, l.department_id, l.email, l.given_name, l.group_id, l.is_di
sabled, l.is_manager, l.local_phone, l.location_address, l.login, l.manager, l.member_of, l.member_of_string, l.mobile_number, l.name, l.position_name, l.postal_code, l.title, l.user_status_code
      FROM ldap_employees AS l
      WHERE l.email IS NOT NULL AND l.email <> ''
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (1ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT e.id, e.dep_id, e.dep_name, e.disabled, e.is_filial, e.is_manager, e.local_phone, e.login, e.login_ad, e.mail, e.manager
_tab_number, e.mobile_phone, e.name, e.parent_dep_id, e.parent_dep_name, e.position, e.status_code, e.status_description, e.tab_number
      FROM public."Employees" AS e
      WHERE e.mail IS NOT NULL AND e.mail <> ''
 Обновлены поля LoginAd в таблице employees.

Process finished with exit code 0.

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

        _logger.LogInformation("Sending paged LDAP search request with filter: {Filter}", _filter);

        var employees = new List<LDAPEmployee>();

        var pageSize = 1000;
        var cookie = Array.Empty<byte>();
        var searchRequest = new SearchRequest(_searchBase, _filter, SearchScope.Subtree, _attributes);

        do
        {
            var pageControl = new PageResultRequestControl(pageSize) { Cookie = cookie };
            searchRequest.Controls.Clear();
            searchRequest.Controls.Add(pageControl);

            var response = (SearchResponse)connection.SendRequest(searchRequest);

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

            var pageResponse = response.Controls
                .OfType<PageResultResponseControl>()
                .FirstOrDefault();

            cookie = pageResponse?.Cookie ?? Array.Empty<byte>();

        } while (cookie != null && cookie.Length > 0);

        _logger.LogInformation("✅ Импортировано {Count} записей из LDAP.", employees.Count);
        return employees;
    }

}

