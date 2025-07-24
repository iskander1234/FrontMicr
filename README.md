// Services/LdapEmployeeSyncService.cs
using System;
using System.Collections.Generic;
using System.DirectoryServices.Protocols;
using System.Linq;
using System.Net;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using DinDin.Models;

namespace DinDin.Services
{
    public class LdapEmployeeSyncService
    {
        private readonly ILogger<LdapEmployeeSyncService> _logger;
        private readonly LdapDirectoryIdentifier        _identifier;
        private readonly NetworkCredential               _credentials;
        private readonly string                          _searchBase;
        private readonly string                          _filter;
        private readonly string[]                        _attributes;
        private const int PageSize = 1000;

        public LdapEmployeeSyncService(IConfiguration configuration, ILogger<LdapEmployeeSyncService> logger)
        {
            _logger     = logger;
            var server  = configuration["LDAPConfig:Server"]!;
            var port    = int.TryParse(configuration["LDAPConfig:Port"], out var p) ? p : 389;
            _identifier = new LdapDirectoryIdentifier(server, port);
            _credentials= new NetworkCredential(
                              configuration["LDAPConfig:BindDN"]!,
                              configuration["LDAPConfig:Password"]!);
            _searchBase = configuration["LDAPConfig:SearchBase"]!;
            _filter     = configuration["LDAPConfig:Filters:UserPerson:Code"]!;
            _attributes = configuration
                              .GetSection("LDAPConfig:Filters:UserPerson:Keys")
                              .Get<string[]>()!;
            _logger.LogInformation("LDAPConfig loaded: {0}:{1}, BaseDN={2}", server, port, _searchBase);
        }

        public List<LDAPEmployee> GetLdapEmployees()
        {
            _logger.LogInformation("Connecting to LDAP {0}...", _identifier);
            using var connection = new LdapConnection(_identifier, _credentials, AuthType.Negotiate);
            connection.SessionOptions.ProtocolVersion = 3;
            connection.Bind();
            _logger.LogInformation("LDAP bind successful");

            var rawEntries = new List<SearchResultEntry>();
            byte[] cookie = Array.Empty<byte>();

            do
            {
                // 1) Базовый запрос
                var req = new SearchRequest(_searchBase, _filter, SearchScope.Subtree, _attributes);

                // 2) Пейджинг (должен идти первым!)
                var pageControl = new PageResultRequestControl(PageSize) { Cookie = cookie };
                req.Controls.Add(pageControl);

                // 3) Сортировка (для детерминированности между страницами)
                var sortControl = new SortRequestControl(new[]
                {
                    // matchingRule = null, reverseOrder = false
                    new SortKey("sAMAccountName", null, false)
                });
                req.Controls.Add(sortControl);

                // 4) Отправляем
                var resp = (SearchResponse)connection.SendRequest(req);
                _logger.LogInformation("Received page: {0} entries; previous cookie length: {1}", resp.Entries.Count, cookie.Length);

                // 5) Сохраняем записи
                foreach (SearchResultEntry e in resp.Entries)
                    rawEntries.Add(e);

                // 6) Обновляем cookie
                cookie = resp.Controls
                            .OfType<PageResultResponseControl>()
                            .FirstOrDefault()?.Cookie
                         ?? Array.Empty<byte>();
            }
            while (cookie.Length > 0);

            _logger.LogInformation("Total raw entries fetched: {0}", rawEntries.Count);

            // 7) Конвертация в модели
            var employees = rawEntries.Select(e => new LDAPEmployee(e)).ToList();
            _logger.LogInformation("Total LDAPEmployee objects created: {0}", employees.Count);
            return employees;
        }
    }
}
