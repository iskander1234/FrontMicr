// ================================
// File: Interfaces/Shared/Repositories/IListRepository.cs
// ================================
namespace DinDin.Interfaces.Shared.Repositories
{
    public interface IListRepository<T>
    {
        /// <summary>
        /// Insert new or update existing entities by the key property (e.g., Login).
        /// </summary>
        Task UpdateByKey(List<T> items);
    }
}


// ================================
// File: Interfaces/Complex/Services/Ldap/IUserLdapService.cs
// ================================
namespace DinDin.Interfaces.Complex.Services.Ldap
{
    public interface IUserLdapService
    {
        /// <summary>
        /// Reads all unique LDAPEmployee records from Active Directory.
        /// </summary>
        List<LDAPEmployee> GetLdapEmployees();
    }
}


// ================================
// File: Models/LDAPEmployee.cs
// (Use your existing model with the two constructors)
// ================================
using System;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;
using System.DirectoryServices.Protocols;
using System.Text.Json;

namespace DinDin.Models
{
    [Table("ldap_employees")]
    public class LDAPEmployee
    {
        [Key] [Column("id")] public int Id { get; set; }
        [Column("user_status_code")] public int UserStatusCode { get; set; }
        [Column("is_disabled")] public bool IsDisabled { get; set; }
        [Column("given_name")] public string? GivenName { get; set; }
        [Column("name")] public string Name { get; set; } = string.Empty;
        [Column("position_name")] public string? PositionName { get; set; }
        [Column("title")] public string? Title { get; set; }
        [Column("department")] public string? Department { get; set; }
        [Column("login")] public string Login { get; set; } = string.Empty;
        [Column("local_phone")] public string? LocalPhone { get; set; }
        [Column("mobile_number")] public string? MobileNumber { get; set; }
        [Column("city")] public string? City { get; set; }
        [Column("location_address")] public string? LocationAddress { get; set; }
        [Column("postal_code")] public string? PostalCode { get; set; }
        [Column("company")] public string? Company { get; set; }
        [Column("manager")] public string? Manager { get; set; }
        [Column("email")] public string? Email { get; set; }
        [Column("member_of")] public string[]? MemberOf { get; set; }
        [Column("member_of_string")] public string? MemberOfString { get; set; }
        [Column("group_id")] public int? GroupId { get; set; }
        [Column("department_id")] public int? DepartmentId { get; set; }
        [Column("department_details")] public JsonDocument? DepartmentDetails { get; set; }
        [Column("is_manager")] public bool IsManager { get; set; }

        public LDAPEmployee() { }

        // From LdapConnection
        public LDAPEmployee(SearchResultEntry entry) { /* ... existing code ... */ }

        // From DirectorySearcher
        public LDAPEmployee(System.DirectoryServices.SearchResult entry) { /* ... existing code ... */ }

        // Helper methods GetString/GetInt omitted for brevity
    }
}


// ================================
// File: Services/LdapEmployeeSyncService.cs
// ================================
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
    public class LdapEmployeeSyncService : IUserLdapService
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
                config["LDAPConfig:BindDN"]!, config["LDAPConfig:Password"]!);
            _searchBase = config["LDAPConfig:SearchBase"]!;
            _filter     = config["LDAPConfig:Filters:UserPerson:Code"] ?? "(objectClass=user)";
            _attributes = config.GetSection("LDAPConfig:Filters:UserPerson:Keys").Get<string[]>()
                          ?? Array.Empty<string>();

            _logger.LogInformation("LDAP initialized: {Server}:{Port}, BaseDN={Base}/", server, port, _searchBase);
        }

        public List<LDAPEmployee> GetLdapEmployees()
        {
            using var conn = new LdapConnection(_identifier, _credentials, AuthType.Negotiate);
            conn.SessionOptions.ProtocolVersion = 3;
            conn.Bind();
            _logger.LogInformation("LDAP bind successful");

            var list = new List<LDAPEmployee>();
            var seen = new HashSet<string>(StringComparer.OrdinalIgnoreCase);
            const string segs = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ";

            foreach (var seg in segs)
            {
                var segFilter = $"(&{_filter}(sAMAccountName={seg}*))";
                _logger.LogInformation("Segment {Seg}", seg);
                byte[] cookie;
                do
                {
                    var req = new SearchRequest(_searchBase, segFilter, SearchScope.Subtree, _attributes);
                    var pgc = new PageResultRequestControl(PageSize) { Cookie = cookie = Array.Empty<byte>() };
                    req.Controls.Add(pgc);

                    var resp = (SearchResponse)conn.SendRequest(req);
                    _logger.LogInformation("Page {Seg}: {Count} entries", seg, resp.Entries.Count);

                    foreach (SearchResultEntry e in resp.Entries)
                    {
                        var login = e.Attributes["sAMAccountName"]?[0]?.ToString();
                        if (login != null && seen.Add(login))
                        {
                            list.Add(new LDAPEmployee(e));
                        }
                    }

                    cookie = resp.Controls.OfType<PageResultResponseControl>().FirstOrDefault()?.Cookie
                              ?? Array.Empty<byte>();
                } while (cookie.Length > 0);
            }

            _logger.LogInformation("Total unique LDAP entries: {Count}", list.Count);
            return list;
        }
    }
}


// ================================
// File: Repositories/LDAPUsersRepository.cs
// ================================
using System.Collections.Generic;
using System.Threading.Tasks;
using DinDin.Interfaces.Shared.Repositories;
using DinDin.Models;
using DinDin.Shared.Contexts;
using EFCore.BulkExtensions;

namespace DinDin.Repositories
{
    public class LDAPUsersRepository : IListRepository<LDAPEmployee>
    {
        private readonly BpmCoreUserContext _ctx;
        public LDAPUsersRepository(BpmCoreUserContext ctx) => _ctx = ctx;

        public async Task UpdateByKey(List<LDAPEmployee> items)
        {
            await _ctx.BulkInsertOrUpdateAsync(items, opt =>
            {
                opt.BatchSize = 1000;
                opt.IncludeGraph = false;
                opt.UpdateByProperties = new[] { "Login" };
                opt.SetOutputIdentity = true;
                opt.UseTempDB = false;
                opt.PropertiesToExclude = new[] { "Id", "GroupId", "DepartmentId", "IsManager" };
            });
        }
    }
}


// ================================
// File: Program.cs
// ================================
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using DinDin.Interfaces.Complex.Services.Ldap;
using DinDin.Interfaces.Shared.Repositories;
using DinDin.Services;
using DinDin.Repositories;
using DinDin.Models;

var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((ctx, sc) =>
    {
        sc.AddSingleton<IUserLdapService, LdapEmployeeSyncService>();
        sc.AddTransient<IListRepository<LDAPEmployee>, LDAPUsersRepository>();
    })
    .Build();

using var scope = host.Services.CreateScope();
var ldapSvc = scope.ServiceProvider.GetRequiredService<IUserLdapService>();
var repo    = scope.ServiceProvider.GetRequiredService<IListRepository<LDAPEmployee>>();

var employees = ldapSvc.GetLdapEmployees();
Console.WriteLine($"Read {employees.Count} LDAP records");
await repo.UpdateByKey(employees);
Console.WriteLine("Sync complete");

await host.RunAsync();
