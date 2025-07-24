using System;
using System.Collections.Generic;
using System.DirectoryServices.Protocols;
using System.Linq;
using System.Net;
using System.Threading.Tasks;
using DinDin.Models; // or DailyDataUpdateApp.Models
using DinDin.Shared.Contexts; // or DailyDataUpdateApp.Shared.Contextes
using EFCore.BulkExtensions;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;

namespace DinDin.Services // or DailyDataUpdateApp.Services
{
    public interface IUserLdapService
    {
        IEnumerable<LDAPEmployee> GetLdapEmployees();
    }

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
            var server = config["LDAPConfig:Server"] ?? throw new ArgumentNullException("LDAPConfig:Server");
            var port = int.TryParse(config["LDAPConfig:Port"], out var p) ? p : 389;
            _identifier = new LdapDirectoryIdentifier(server, port);
            _credentials = new NetworkCredential(
                config["LDAPConfig:BindDN"]!,
                config["LDAPConfig:Password"]!
            );
            _searchBase = config["LDAPConfig:SearchBase"]!;
            _filter = config["LDAPConfig:Filters:UserPerson:Code"] ?? "(objectClass=user)";
            _attributes = config.GetSection("LDAPConfig:Filters:UserPerson:Keys").Get<string[]>()!;

            _logger.LogInformation("LDAP initialized: {Server}:{Port} Base={Base}", server, port, _searchBase);
        }

        public IEnumerable<LDAPEmployee> GetLdapEmployees()
        {
            using var connection = new LdapConnection(_identifier, _credentials, AuthType.Negotiate);
            connection.SessionOptions.ProtocolVersion = 3;
            connection.Bind();
            _logger.LogInformation("LDAP bind successful");

            var employees = new List<LDAPEmployee>();
            var seenLogins = new HashSet<string>(StringComparer.OrdinalIgnoreCase);
            const string segments = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ";

            foreach (var seg in segments)
            {
                var segFilter = $"(&{_filter}(sAMAccountName={seg}*))";
                _logger.LogInformation("Segment: {Seg}", seg);
                byte[] cookie = Array.Empty<byte>();

                do
                {
                    var req = new SearchRequest(_searchBase, segFilter, SearchScope.Subtree, _attributes);
                    var pageCtrl = new PageResultRequestControl(PageSize) { Cookie = cookie };
                    req.Controls.Add(pageCtrl);

                    var resp = (SearchResponse)connection.SendRequest(req);
                    _logger.LogInformation("Loaded {Count} entries in segment {Seg}", resp.Entries.Count, seg);

                    foreach (SearchResultEntry e in resp.Entries)
                    {
                        var login = e.Attributes["sAMAccountName"]?[0]?.ToString() ?? string.Empty;
                        if (login == string.Empty || !seenLogins.Add(login))
                            continue;
                        employees.Add(new LDAPEmployee(e));
                    }

                    cookie = resp.Controls
                        .OfType<PageResultResponseControl>()
                        .FirstOrDefault()?.Cookie
                        ?? Array.Empty<byte>();
                }
                while (cookie.Length > 0);
            }

            _logger.LogInformation("Total unique LDAP entries: {Count}", employees.Count);
            return employees;
        }
    }
}

namespace DinDin.Repositories // or DailyDataUpdateApp.Repositories
{
    public class LDAPUsersRepository : IListRepository<LDAPEmployee>
    {
        private readonly BpmCoreUserContext _context;

        public LDAPUsersRepository(BpmCoreUserContext context)
        {
            _context = context;
        }

        public async Task UpdateByKey(List<LDAPEmployee> updatedLdapEmployees)
        {
            await _context.BulkInsertOrUpdateAsync(updatedLdapEmployees, options =>
            {
                options.BatchSize = 1000;
                options.IncludeGraph = false;
                options.UpdateByProperties = new[] { "Login" };
                options.SetOutputIdentity = true;
                options.UseTempDB = false;
                options.PropertiesToExclude = new[] { "Id", "GroupId", "DepartmentId", "IsManager" };
            });
        }
    }
}
