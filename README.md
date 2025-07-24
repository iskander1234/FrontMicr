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
        private readonly string[]                         _attributes;
        private readonly string                           _bindDn;
        private readonly string                           _password;
        private readonly string                           _searchBase;
        private readonly string                           _filter;
        private readonly string                           _server;
        private readonly int                              _port;

        public LdapEmployeeSyncService(
            IConfiguration configuration,
            ILogger<LdapEmployeeSyncService> logger)
        {
            _logger     = logger;
            _server     = configuration["LDAPConfig:Server"]!;
            _port       = int.Parse(configuration["LDAPConfig:Port"]!);
            _bindDn     = configuration["LDAPConfig:BindDN"]!;
            _password   = configuration["LDAPConfig:Password"]!;
            _searchBase = configuration["LDAPConfig:SearchBase"]!;
            _filter     = configuration["LDAPConfig:Filters:UserPerson:Code"]!;
            _attributes = configuration
                              .GetSection("LDAPConfig:Filters:UserPerson:Keys")
                              .Get<string[]>()!;
        }

        public List<LDAPEmployee> GetLdapEmployees()
        {
            _logger.LogInformation("Connecting to LDAP {Server}:{Port}, base \"{Base}\"", _server, _port, _searchBase);

            var credential = new NetworkCredential(_bindDn, _password);
            var identifier = new LdapDirectoryIdentifier(_server, _port);
            using var connection = new LdapConnection(identifier, credential, AuthType.Negotiate);
            connection.SessionOptions.ProtocolVersion = 3;
            connection.Bind();
            _logger.LogInformation("LDAP bind successful");

            var employees = new List<LDAPEmployee>();
            byte[] cookie = Array.Empty<byte>();
            const int pageSize = 1000;

            do
            {
                var request = new SearchRequest(
                    _searchBase,
                    _filter,
                    SearchScope.Subtree,
                    _attributes
                );

                // создаём fresh PageResultRequestControl на каждую итерацию
                var pageControl = new PageResultRequestControl(pageSize) { Cookie = cookie };
                request.Controls.Add(pageControl);

                var response = (SearchResponse)connection.SendRequest(request);

                _logger.LogInformation("Fetched {Count} entries; previous cookie length: {CookieLen}",
                    response.Entries.Count,
                    cookie.Length);

                foreach (SearchResultEntry entry in response.Entries)
                {
                    try
                    {
                        employees.Add(new LDAPEmployee(entry));
                    }
                    catch (Exception ex)
                    {
                        _logger.LogWarning(ex, "Failed to parse entry {DN}", entry.DistinguishedName);
                    }
                }

                // получаем новую куку, если она есть
                cookie = response.Controls
                    .OfType<PageResultResponseControl>()
                    .FirstOrDefault()
                    ?.Cookie
                        ?? Array.Empty<byte>();

            } while (cookie.Length > 0);

            _logger.LogInformation("🎯 Total records retrieved from LDAP: {Total}", employees.Count);
            return employees;
        }
    }
}
