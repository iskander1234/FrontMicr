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

            _logger.LogInformation("LDAP Config loaded: Server={Server}, Port={Port}, BindDN={BindDN}, SearchBase={SearchBase}",
                _server, _port, _bindDn, _searchBase);
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
            const int pageSize = 1000;
            byte[] cookie = Array.Empty<byte>();

            // Сортировка для стабильной пагинации
            var sortKey = new SortKey("sAMAccountName");
            var sortControl = new SortRequestControl(new[] { sortKey });
            var pageControl = new PageResultRequestControl(pageSize);

            do
            {
                var request = new SearchRequest(_searchBase, _filter, SearchScope.Subtree, _attributes);
                request.Controls.Add(sortControl);

                pageControl.Cookie = cookie;
                request.Controls.Add(pageControl);

                var response = (SearchResponse)connection.SendRequest(request);

                foreach (SearchResultEntry entry in response.Entries)
                {
                    try
                    {
                        employees.Add(new LDAPEmployee(entry));
                    }
                    catch (Exception ex)
                    {
                        _logger.LogWarning(ex, "Failed to parse LDAP entry");
                    }
                }

                cookie = response.Controls
                    .OfType<PageResultResponseControl>()
                    .FirstOrDefault()?
                    .Cookie
                    ?? Array.Empty<byte>();

                _logger.LogInformation("Loaded {Count} entries, total so far: {Total}", response.Entries.Count, employees.Count);

            } while (cookie.Length > 0);

            _logger.LogInformation("🎯 Total records retrieved from LDAP: {Count}", employees.Count);
            return employees;
        }
    }
}
