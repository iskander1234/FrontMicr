using System;
using System.Collections.Generic;
// Ensure you have installed the NuGet package:
// <PackageReference Include="System.DirectoryServices.Protocols" Version="8.0.0" />
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
            connection.Bind();
            _logger.LogInformation("LDAP bind successful");

            var employees = new List<LDAPEmployee>();
            // Уникальность по полю Login
            var seenLogins = new HashSet<string>(StringComparer.OrdinalIgnoreCase);
            const int pageSize = 1000;
            byte[] cookie = Array.Empty<byte>();

            // Сегментация по префиксу sAMAccountName: цифры, латиница верхнего и нижнего регистра
            const string segments = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";

            foreach (var segment in segments)
            {
                var segmentFilter = $"(&{_filter}(sAMAccountName={segment}*))";
                _logger.LogInformation("Segment filter: {Filter}", segmentFilter);

                cookie = Array.Empty<byte>();
                var pageControl = new PageResultRequestControl(pageSize);

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
                            _logger.LogWarning(ex, "Failed to parse entry for segment '{Segment}'", segment);
                        }
                    }

                    cookie = response.Controls
                        .OfType<PageResultResponseControl>()
                        .FirstOrDefault()?
                        .Cookie
                        ?? Array.Empty<byte>();

                    _logger.LogInformation("Segment {Segment}: loaded {Count} entries, total so far: {Total}",
                        segment, response.Entries.Count, employees.Count);

                } while (cookie.Length > 0);
            }

            // Удаляем дубликаты по логину
            _logger.LogInformation("🔄 Removing duplicates by Login... original count: {OrigCount}", employees.Count);
            var uniqueEmployees = employees
                .Where(e => !string.IsNullOrEmpty(e.Login) && seenLogins.Add(e.Login))
                .ToList();
            _logger.LogInformation("✅ Count after deduplication: {UniqueCount}", uniqueEmployees.Count);
            _logger.LogInformation("🎯 Total unique records retrieved from LDAP: {Count}", uniqueEmployees.Count);
            return uniqueEmployees;
        }
