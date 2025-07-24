"C:\Program Files\JetBrains\JetBrains Rider 2025.1.2\plugins\dpa\DotFiles\JetBrains.DPA.Runner.exe" --handle=1564 --backend-pid=8868 --etw-collect-flags=67108622 --detach-event-name=dpa.detach.8868.94 --refresh-interval=1 -- C:/BPM/Leshan/1/DinDin/bin/Debug/net8.0/DinDin.exe
warn: Microsoft.EntityFrameworkCore.Model.Validation[10400]
      Sensitive data logging is enabled. Log entries and exception messages may include sensitive application data; this mode should only be enabled during development.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (173ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      CREATE TABLE "public"."DepartmentsTempc0f700ab" AS TABLE "public"."Departments" WITH NO DATA;
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (48ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO "public"."Departments" ("id", "actual", "manager_id", "name", "parent_id") (SELECT "id", "actual", "manager_id", "name", "parent_id" FROM "public"."DepartmentsTempc0f700ab") ON CONFLICT ("id") DO UPDATE SET "actual" = EXCLUDED."actual", "manager_id" = EXCLUDED."manager_id", "name" = EXCLUDED."name", "parent_id" = EXCLUDED."parent_id" RETURNING "public"."Departments"."id", "public"."Departments"."actual", "public"."Departments"."manager_id", "public"."Departments"."name", "public"."Departments"."parent_id"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (20ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      DROP TABLE IF EXISTS "public"."DepartmentsTempc0f700ab"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (14ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      CREATE TABLE "public"."DepartmentsTempb28bf9bb" AS TABLE "public"."Departments" WITH NO DATA;
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (29ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO "public"."Departments" ("id", "actual", "manager_id", "name", "parent_id") (SELECT "id", "actual", "manager_id", "name", "parent_id" FROM "public"."DepartmentsTempb28bf9b
b") ON CONFLICT ("id") DO UPDATE SET "actual" = EXCLUDED."actual", "manager_id" = EXCLUDED."manager_id", "name" = EXCLUDED."name", "parent_id" = EXCLUDED."parent_id" RETURNING "public"."Departments"."id", "public"."Departments"."actual", "public"."Departments"."manager_id", "public"."Departments"."name", "public"."Departments"."parent_id"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (2ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      DROP TABLE IF EXISTS "public"."DepartmentsTempb28bf9bb"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (24ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT e.id, e.dep_id, e.dep_name, e.disabled, e.is_filial, e.is_manager, e.local_phone, e.login, e.login_ad, e.mail, e.manager_tab_number, e.mobile_phone, e.name, e.parent_dep_id, e.parent_dep_name, e.position, e.status_code, e.status_description, e.tab_number
      FROM public."Employees" AS e
info: DinDin.Services.LdapEmployeeSyncService[0]
      LDAP Config loaded: Server=172.31.0.252, Port=389, BindDN=gnpfadm@enpf.kz, SearchBase=dc=enpf,dc=kz
info: DinDin.Services.LdapEmployeeSyncService[0]
      Connecting to LDAP via DirectorySearcher: 172.31.0.252:389...
info: DinDin.Services.LdapEmployeeSyncService[0]
      Starting DirectorySearcher.FindAll()...
info: DinDin.Services.LdapEmployeeSyncService[0]
      DirectorySearcher returned 4233 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      ?? Total records retrieved from LDAP: 0
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (45ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      TRUNCATE TABLE "ldap_employees" RESTART IDENTITY
 Импортировано 0 записей в таблицу ldap_employees.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (5ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT l.id, l.city, l.company, l.department, l.department_details, l.department_id, l.email, l.given_name, l.group_id, l.is_disabled, l.is_manager, l.local_phone, l.location_address, l.login, l.manager, l.member_of, l.member_of_string, l.mobile_number, l.name, l.position_name, l.postal_code, l.title, l.user_status_code
      FROM ldap_employees AS l
      WHERE l.email IS NOT NULL AND l.email <> ''
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (3ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT e.id, e.dep_id, e.dep_name, e.disabled, e.is_filial, e.is_manager, e.local_phone, e.login, e.login_ad, e.mail, e.manager_tab_number, e.mobile_phone, e.name, e.parent_dep_id, e.parent_dep_name, e.position, e.status_code, e.status_description, e.tab_number
      FROM public."Employees" AS e
      WHERE e.mail IS NOT NULL AND e.mail <> ''
 Обновлены поля LoginAd в таблице employees.

Process finished with exit code 0.

using System;
using System.Collections.Generic;
using System.DirectoryServices;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using DinDin.Models;

namespace DinDin.Services
{
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
            _server = configuration["LDAPConfig:Server"] ?? throw new ArgumentNullException("LDAPConfig:Server is required");
            _port = int.TryParse(configuration["LDAPConfig:Port"], out var port) ? port : 389;
            _bindDn = configuration["LDAPConfig:BindDN"] ?? throw new ArgumentNullException("LDAPConfig:BindDN is required");
            _password = configuration["LDAPConfig:Password"] ?? throw new ArgumentNullException("LDAPConfig:Password is required");
            _searchBase = configuration["LDAPConfig:SearchBase"] ?? throw new ArgumentNullException("LDAPConfig:SearchBase is required");
            _filter = configuration["LDAPConfig:Filters:UserPerson:Code"] ?? "(objectClass=user)";
            _attributes = configuration.GetSection("LDAPConfig:Filters:UserPerson:Keys").Get<string[]>() ?? Array.Empty<string>();

            _logger.LogInformation("LDAP Config loaded: Server={Server}, Port={Port}, BindDN={BindDN}, SearchBase={SearchBase}",
                _server, _port, _bindDn, _searchBase);
        }

        public List<LDAPEmployee> GetLdapEmployees()
        {
            _logger.LogInformation("Connecting to LDAP via DirectorySearcher: {Server}:{Port}...", _server, _port);
            var employees = new List<LDAPEmployee>();

            try
            {
                using var entry = new DirectoryEntry(
                    $"LDAP://{_server}:{_port}/{_searchBase}",
                    _bindDn,
                    _password,
                    AuthenticationTypes.Secure);

                using var searcher = new DirectorySearcher(entry)
                {
                    Filter = _filter,
                    PageSize = 1000,
                    ReferralChasing = ReferralChasingOption.All,
                    SearchScope = SearchScope.Subtree,
                    CacheResults = false
                };

                foreach (var attr in _attributes)
                    searcher.PropertiesToLoad.Add(attr);

                _logger.LogInformation("Starting DirectorySearcher.FindAll()...");
                using var results = searcher.FindAll();
                _logger.LogInformation("DirectorySearcher returned {Count} entries", results.Count);

                foreach (SearchResult res in results)
                {
                    try
                    {
                        employees.Add(new LDAPEmployee(res));
                    }
                    catch (Exception ex)
                    {
                        _logger.LogWarning(ex, "Failed to parse entry {Path}", res.Path);
                    }
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "DirectorySearcher failed: {Message}", ex.Message);
                throw;
            }

            _logger.LogInformation("🎯 Total records retrieved from LDAP: {Count}", employees.Count);
            return employees;
        }
    }
}




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

        [Column("member_of_string", TypeName = "varchar")]
        public string? MemberOfString { get; set; }

        [Column("department_id")]
        public int? DepartmentId { get; set; }

        [Column("department_details")]
        public JsonDocument? DepartmentDetails { get; set; }

        [Column("group_id")]
        public int? GroupId { get; set; }

        [Column("is_manager")]
        public bool IsManager { get; set; }

        public LDAPEmployee() { }

        // Constructor for LdapConnection (SearchResultEntry)
        public LDAPEmployee(SearchResultEntry entry)
        {
            UserStatusCode = entry.GetInt("userAccountControl");
            IsDisabled = (UserStatusCode & 2) != 0;
            GivenName = entry.GetString("givenName");
            Name = entry.GetString("cn") ?? string.Empty;
            PositionName = entry.GetString("description");
            Title = entry.GetString("title");
            Department = entry.GetString("department");
            Login = entry.GetString("sAMAccountName") ?? string.Empty;
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

        // New constructor for DirectorySearcher (SearchResult)
        public LDAPEmployee(SearchResult entry)
        {
            UserStatusCode = GetInt(entry, "userAccountControl");
            IsDisabled = (UserStatusCode & 2) != 0;
            GivenName = GetString(entry, "givenName");
            Name = GetString(entry, "cn") ?? string.Empty;
            PositionName = GetString(entry, "description");
            Title = GetString(entry, "title");
            Department = GetString(entry, "department");
            Login = GetString(entry, "sAMAccountName") ?? string.Empty;
            LocalPhone = GetString(entry, "telephoneNumber");
            MobileNumber = GetString(entry, "mobile");
            City = GetString(entry, "l");
            LocationAddress = GetString(entry, "streetAddress");
            PostalCode = GetString(entry, "postalCode");
            Company = GetString(entry, "company");
            Manager = GetString(entry, "manager");
            Email = GetString(entry, "mail");
            if (entry.Properties.Contains("memberOf"))
            {
                MemberOf = entry.Properties["memberOf"].Cast<object>()
                                .Select(o => o.ToString()!)
                                .ToArray();
                MemberOfString = string.Join(";", MemberOf);
            }
            else
            {
                MemberOf = Array.Empty<string>();
                MemberOfString = null;
            }
            IsManager = false;
        }

        private static string? GetString(SearchResult entry, string propName)
        {
            if (entry.Properties.Contains(propName) && entry.Properties[propName].Count > 0)
                return entry.Properties[propName][0]?.ToString();
            return null;
        }

        private static int GetInt(SearchResult entry, string propName)
        {
            var s = GetString(entry, propName);
            return int.TryParse(s, out var i) ? i : 0;
        }
    }



