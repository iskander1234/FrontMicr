using System;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;
using System.DirectoryServices.Protocols;
using System.Linq;
using System.Text.Json;

namespace DinDin.Models
{
    [Table("ldap_employees")]
    public class LDAPEmployee
    {
        [Key] [Column("id")]                public int    Id                { get; set; }
        [Column("user_status_code")]        public int    UserStatusCode    { get; set; }
        [Column("is_disabled")]             public bool   IsDisabled        { get; set; }
        [Column("given_name")]              public string?GivenName         { get; set; }
        [Column("name")]                    public string Name              { get; set; } = "";
        [Column("position_name")]           public string?PositionName     { get; set; }
        [Column("title")]                   public string?Title            { get; set; }
        [Column("department")]              public string?Department       { get; set; }
        [Column("login")]                   public string Login             { get; set; } = "";
        [Column("local_phone")]             public string?LocalPhone       { get; set; }
        [Column("mobile_number")]           public string?MobileNumber     { get; set; }
        [Column("city")]                    public string?City             { get; set; }
        [Column("location_address")]        public string?LocationAddress  { get; set; }
        [Column("postal_code")]             public string?PostalCode       { get; set; }
        [Column("company")]                 public string?Company          { get; set; }
        [Column("manager")]                 public string?Manager          { get; set; }
        [Column("email")]                   public string?Email            { get; set; }
        [Column("member_of")]               public string[]?MemberOf       { get; set; }
        [Column("member_of_string")]        public string?MemberOfString   { get; set; }
        [Column("group_id")]                public int?    GroupId          { get; set; }
        [Column("department_id")]           public int?    DepartmentId     { get; set; }
        [Column("department_details")]      public JsonDocument? DepartmentDetails { get; set; }
        [Column("is_manager")]              public bool   IsManager         { get; set; }

        public LDAPEmployee() { }

        // Конструктор для LdapConnection (SearchResultEntry)
        public LDAPEmployee(SearchResultEntry entry)
        {
            string? getStr(string attr) =>
                entry.Attributes.Contains(attr) && entry.Attributes[attr].Count > 0
                  ? entry.Attributes[attr][0]?.ToString()
                  : null;

            UserStatusCode = int.TryParse(getStr("userAccountControl"), out var usc) ? usc : 0;
            IsDisabled     = (UserStatusCode & 2) != 0;
            GivenName      = getStr("givenName");
            Name           = getStr("cn") ?? "";
            PositionName   = getStr("description");
            Title          = getStr("title");
            Department     = getStr("department");
            Login          = getStr("sAMAccountName") ?? "";
            LocalPhone     = getStr("telephoneNumber");
            MobileNumber   = getStr("mobile");
            City           = getStr("l");
            LocationAddress= getStr("streetAddress");
            PostalCode     = getStr("postalCode");
            Company        = getStr("company");
            Manager        = getStr("manager");
            Email          = getStr("mail");
            if (entry.Attributes.Contains("memberOf"))
            {
                MemberOf = entry
                  .Attributes["memberOf"]
                  .Cast<object>()
                  .Select(o => o.ToString()!)
                  .ToArray();
                MemberOfString = string.Join(";", MemberOf);
            }
            else
            {
                MemberOf = Array.Empty<string>();
                MemberOfString = null;
            }
        }
    }
}


using System;
using System.Collections.Generic;
using System.DirectoryServices.Protocols;
using System.Linq;
using System.Net;
using DinDin.Models;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;

namespace DinDin.Services
{
    public class LdapEmployeeSyncService
    {
        private readonly ILogger<LdapEmployeeSyncService> _logger;
        private readonly LdapDirectoryIdentifier _identifier;
        private readonly NetworkCredential _credentials;
        private readonly string _searchBase;
        private readonly string _filter;
        private readonly string[] _attributes;
        private const int PageSize = 1000;

        public LdapEmployeeSyncService(IConfiguration config, ILogger<LdapEmployeeSyncService> logger)
        {
            _logger = logger;
            var server = config["LDAPConfig:Server"]!;
            var port   = int.TryParse(config["LDAPConfig:Port"], out var p) ? p : 389;
            _identifier = new LdapDirectoryIdentifier(server, port);
            _credentials = new NetworkCredential(
                config["LDAPConfig:BindDN"]!,
                config["LDAPConfig:Password"]!
            );
            _searchBase = config["LDAPConfig:SearchBase"]!;
            _filter     = config["LDAPConfig:Filters:UserPerson:Code"] ?? "(objectClass=user)";
            _attributes = config
                .GetSection("LDAPConfig:Filters:UserPerson:Keys")
                .Get<string[]>() ?? Array.Empty<string>();

            _logger.LogInformation("LDAP initialized for {0}:{1}, BaseDN={2}", server, port, _searchBase);
        }

        public List<LDAPEmployee> GetLdapEmployees()
        {
            using var conn = new LdapConnection(_identifier, _credentials, AuthType.Negotiate);
            conn.SessionOptions.ProtocolVersion = 3;
            conn.Bind();
            _logger.LogInformation("LDAP bind successful");

            var employees = new List<LDAPEmployee>();
            var seenLogins = new HashSet<string>(StringComparer.OrdinalIgnoreCase);
            const string segments = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ";

            foreach (var seg in segments)
            {
                var segFilter = $"(&{_filter}(sAMAccountName={seg}*))";
                _logger.LogInformation("Segment {0} filter: {1}", seg, segFilter);
                byte[] cookie = Array.Empty<byte>();

                do
                {
                    var req = new SearchRequest(_searchBase, segFilter, SearchScope.Subtree, _attributes);
                    var pgc = new PageResultRequestControl(PageSize) { Cookie = cookie };
                    req.Controls.Add(pgc);

                    var resp = (SearchResponse)conn.SendRequest(req);
                    _logger.LogInformation("Segment {0}: loaded {1} entries", seg, resp.Entries.Count);

                    foreach (SearchResultEntry e in resp.Entries)
                    {
                        var login = e.Attributes["sAMAccountName"]?[0]?.ToString() ?? "";
                        if (login != "" && seenLogins.Add(login))
                        {
                            employees.Add(new LDAPEmployee(e));
                        }
                    }

                    cookie = resp.Controls
                        .OfType<PageResultResponseControl>()
                        .FirstOrDefault()?.Cookie ?? Array.Empty<byte>();
                }
                while (cookie.Length > 0);
            }

            _logger.LogInformation("🎯 Total unique records: {0}", employees.Count);
            return employees;
        }
    }
}


using System.Collections.Generic;
using System.Threading.Tasks;
using DinDin.Models;
using DinDin.Shared.Contexts;   // твой BpmcoreContext
using EFCore.BulkExtensions;

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
        public async Task ReplaceAllLdapEmployees(List<LDAPEmployee> employees)
        {
            await _ctx.Database.ExecuteSqlRawAsync("TRUNCATE TABLE \"ldap_employees\" RESTART IDENTITY");
            var cfg = new BulkConfig { BatchSize = 1000, UseTempDB = true };
            await _ctx.BulkInsertAsync(employees, cfg);
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



.AddSingleton<LdapEmployeeSyncService>()
.AddSingleton<LDAPUsersRepository>()



private static async Task SyncLdap(IServiceProvider provider)
{
    var ldapService   = provider.GetRequiredService<LdapEmployeeSyncService>();
    var ldapUsersRepo = provider.GetRequiredService<LDAPUsersRepository>();

    var allEmployees = ldapService.GetLdapEmployees();
    Console.WriteLine($"🔄 Retrieved {allEmployees.Count} LDAP records");

    await ldapUsersRepo.ReplaceAllLdapEmployees(allEmployees);
    Console.WriteLine($"✅ Imported {allEmployees.Count} into ldap_employees table");

    await ldapUsersRepo.SyncEmployeeLoginsByEmail();
    Console.WriteLine("🔄 Updated LoginAd in Employees table");
}
