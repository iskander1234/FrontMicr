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

        public LdapEmployeeSyncService(IConfiguration configuration, ILogger<LdapEmployeeSyncService> logger)
        {
            _logger = logger;
            var server = configuration["LDAPConfig:Server"] ?? throw new ArgumentNullException("LDAPConfig:Server is required");
            var port = int.TryParse(configuration["LDAPConfig:Port"], out var p) ? p : 389;
            _identifier = new LdapDirectoryIdentifier(server, port);
            _credentials = new NetworkCredential(
                configuration["LDAPConfig:BindDN"] ?? throw new ArgumentNullException("LDAPConfig:BindDN is required"),
                configuration["LDAPConfig:Password"] ?? throw new ArgumentNullException("LDAPConfig:Password is required")
            );
            _searchBase = configuration["LDAPConfig:SearchBase"] ?? throw new ArgumentNullException("LDAPConfig:SearchBase is required");
            _filter = configuration["LDAPConfig:Filters:UserPerson:Code"] ?? "(objectClass=user)";
            _attributes = configuration.GetSection("LDAPConfig:Filters:UserPerson:Keys").Get<string[]>() ?? Array.Empty<string>();

            _logger.LogInformation("LDAP service configured for {Server}:{Port}, BaseDN={BaseDN}", server, port, _searchBase);
        }

        public List<LDAPEmployee> GetLdapEmployees()
        {
            using var connection = new LdapConnection(_identifier, _credentials, AuthType.Negotiate);
            connection.SessionOptions.ProtocolVersion = 3;
            connection.Bind();
            _logger.LogInformation("LDAP bind successful");

            var employees = new List<LDAPEmployee>();
            var seenLogins = new HashSet<string>(StringComparer.OrdinalIgnoreCase);

            // Segmentation prefixes: digits and uppercase letters
            const string segments = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ";

            foreach (var segment in segments)
            {
                var segmentFilter = $"(&{_filter}(sAMAccountName={segment}*))";
                _logger.LogInformation("Segment filter: {Filter}", segmentFilter);
                byte[] cookie = Array.Empty<byte>();

                do
                {
                    var request = new SearchRequest(_searchBase, segmentFilter, SearchScope.Subtree, _attributes);
                    var pageControl = new PageResultRequestControl(PageSize) { Cookie = cookie };
                    request.Controls.Add(pageControl);

                    var response = (SearchResponse)connection.SendRequest(request);
                    _logger.LogInformation("Segment {Segment}: loaded {Count} entries", segment, response.Entries.Count);

                    foreach (SearchResultEntry entry in response.Entries)
                    {
                        var login = entry.Attributes["sAMAccountName"]?[0]?.ToString() ?? string.Empty;
                        if (string.IsNullOrEmpty(login) || !seenLogins.Add(login))
                            continue;
                        try
                        {
                            employees.Add(new LDAPEmployee(entry));
                        }
                        catch (Exception ex)
                        {
                            _logger.LogWarning(ex, "Failed to parse entry with login {Login}", login);
                        }
                    }

                    cookie = response.Controls.OfType<PageResultResponseControl>().FirstOrDefault()?.Cookie ?? Array.Empty<byte>();
                }
                while (cookie.Length > 0);
            }

            _logger.LogInformation("🎯 Total unique records retrieved: {Count}", employees.Count);
            return employees;
        }
    }
}
