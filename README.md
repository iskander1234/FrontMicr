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

            _logger.LogInformation("LDAP Config loaded: Server={Server}, Port={Port}, BindDN={BindDN}, SearchBase={SearchBase}", _server, _port, _bindDn, _searchBase);
        }

        public List<LDAPEmployee> GetLdapEmployees()
        {
            _logger.LogInformation("Connecting to LDAP server {Server}:{Port}...", _server, _port);

            var credentials = new NetworkCredential(_bindDn, _password);
            var identifier = new LdapDirectoryIdentifier(_server, _port);

            using var connection = new LdapConnection(identifier, credentials, AuthType.Negotiate);
            connection.SessionOptions.ProtocolVersion = 3;
            connection.Bind();

            var employees = new List<LDAPEmployee>();
            const int pageSize = 1000;

            // Разбиение по первой букве sAMAccountName
            var segments = Enumerable.Range('0', '9' - '0' + 1)
                             .Select(i => (char)i)
                             .Concat(Enumerable.Range('A', 'Z' - 'A' + 1).Select(i => (char)i))
                             .ToList();

            foreach (var startChar in segments)
            {
                char endChar = (char)(startChar + 1);
                string segmentFilter = startChar == 'Z'
                    ? $"(&{_filter}(sAMAccountName>={startChar}))"
                    : $"(&{_filter}(sAMAccountName>={startChar})(sAMAccountName<{endChar}))";

                _logger.LogInformation("Starting segment filter: {Filter}", segmentFilter);

                var pageControl = new PageResultRequestControl(pageSize);
                byte[] cookie = Array.Empty<byte>();

                do
                {
                    var request = new SearchRequest(_searchBase, segmentFilter, SearchScope.Subtree, _attributes);
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
                            _logger.LogWarning(ex, "Failed to parse entry in segment {Segment}", startChar);
                        }
                    }

                    var pageResp = response.Controls.OfType<PageResultResponseControl>().FirstOrDefault();
                    cookie = pageResp?.Cookie ?? Array.Empty<byte>();

                    _logger.LogInformation("Segment {Segment}: Loaded {Count} records, total so far: {Total}", startChar, response.Entries.Count, employees.Count);

                } while (cookie != null && cookie.Length > 0);
            }

            _logger.LogInformation("🎯 Total records retrieved from LDAP: {Count}", employees.Count);
            return employees;
        }
    }
}
