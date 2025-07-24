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
        private readonly int    _port;
        private readonly string _bindDn;
        private readonly string _password;
        private readonly string _searchBase;
        private readonly string _filter;
        private readonly string[] _attributes;

        public LdapEmployeeSyncService(IConfiguration configuration, ILogger<LdapEmployeeSyncService> logger)
        {
            _logger    = logger;
            _server    = configuration["LDAPConfig:Server"]    ?? throw new ArgumentNullException("Server");
            _port      = int.TryParse(configuration["LDAPConfig:Port"], out var p) ? p : 389;
            _bindDn    = configuration["LDAPConfig:BindDN"]    ?? throw new ArgumentNullException("BindDN");
            _password  = configuration["LDAPConfig:Password"]  ?? throw new ArgumentNullException("Password");
            _searchBase= configuration["LDAPConfig:SearchBase"]?? throw new ArgumentNullException("SearchBase");
            _filter    = configuration["LDAPConfig:Filters:UserPerson:Code"]
                         ?? "(objectClass=user)";
            _attributes= configuration.GetSection("LDAPConfig:Filters:UserPerson:Keys")
                         .Get<string[]>() ?? Array.Empty<string>();

            _logger.LogInformation(
              "LDAP Config loaded: Server={Server}, Port={Port}, BindDN={BindDN}, SearchBase={SearchBase}",
               _server, _port, _bindDn, _searchBase);
        }

        public List<LDAPEmployee> GetLdapEmployees()
        {
            _logger.LogInformation("Connecting to LDAP server {Server}:{Port}...", _server, _port);

            var creds      = new NetworkCredential(_bindDn, _password);
            var identifier = new LdapDirectoryIdentifier(_server, _port);
            using var connection = new LdapConnection(identifier, creds, AuthType.Negotiate);
            connection.SessionOptions.ProtocolVersion = 3;
            connection.Bind();
            _logger.LogInformation("LDAP bind successful");

            var employees = new List<LDAPEmployee>();
            byte[] cookie = Array.Empty<byte>();
            const int pageSize = 1000;

            do
            {
                // каждый раз создаём новый PageResultRequestControl
                var pageControl = new PageResultRequestControl(pageSize) { Cookie = cookie };

                var request = new SearchRequest(
                    _searchBase,
                    _filter,
                    SearchScope.Subtree,
                    _attributes);

                request.Controls.Add(pageControl);

                var response = (SearchResponse)connection.SendRequest(request);

                // добавляем все записи текущей страницы
                foreach (SearchResultEntry entry in response.Entries)
                {
                    try
                    {
                        employees.Add(new LDAPEmployee(entry));
                    }
                    catch (Exception ex)
                    {
                        _logger.LogWarning(
                          ex,
                          "Failed to parse entry {DN}",
                          entry.DistinguishedName);
                    }
                }

                // обновляем cookie — если пусто, значит страниц больше нет
                cookie = response.Controls
                                 .OfType<PageResultResponseControl>()
                                 .FirstOrDefault()?.Cookie
                       ?? Array.Empty<byte>();

                _logger.LogInformation(
                  "Fetched page: {Count} entries; total so far: {Total}",
                  response.Entries.Count,
                  employees.Count);

                // если LDAP сервер жалуется на слишком быстрые запросы — можно раскомментировать:
                // System.Threading.Thread.Sleep(100);

            } while (cookie.Length > 0);

            _logger.LogInformation("🎯 Total records retrieved from LDAP: {Count}", employees.Count);
            return employees;
        }
    }
}
