// File: Services/LdapEmployeeSyncService.cs
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

            _logger.LogInformation("LDAPConfig: {0}:{1}, BaseDN={2}", server, port, _searchBase);
        }

        public List<LDAPEmployee> GetLdapEmployees()
        {
            _logger.LogInformation("Connecting to LDAP {0}...", _identifier);
            using var connection = new LdapConnection(_identifier, _credentials, AuthType.Negotiate);
            connection.SessionOptions.ProtocolVersion = 3;
            connection.Bind();
            _logger.LogInformation("LDAP bind successful");

            var entries = new List<SearchResultEntry>();
            byte[] cookie = Array.Empty<byte>();

            do
            {
                // 1) Формируем запрос
                var request = new SearchRequest(
                    _searchBase,
                    _filter,
                    SearchScope.Subtree,
                    _attributes);

                // 2) Сортировка по sAMAccountName (чтобы результаты не меняться между страницами)
                var sortControl = new SortRequestControl(new[]
                {
                    new SortKey("sAMAccountName", null, false)
                });
                request.Controls.Add(sortControl);

                // 3) Пагинация: создаём новый контрол на каждый запрос
                var pageControl = new PageResultRequestControl(PageSize) { Cookie = cookie };
                request.Controls.Add(pageControl);

                // 4) Отправляем и получаем страницу
                var response = (SearchResponse)connection.SendRequest(request);
                _logger.LogInformation("Received page: {0} entries", response.Entries.Count);

                // 5) Сохраняем
                foreach (SearchResultEntry entry in response.Entries)
                    entries.Add(entry);

                // 6) Получаем новый cookie
                cookie = response.Controls
                            .OfType<PageResultResponseControl>()
                            .FirstOrDefault()?.Cookie
                         ?? Array.Empty<byte>();
            }
            while (cookie.Length > 0);

            _logger.LogInformation("Total LDAP entries retrieved: {0}", entries.Count);

            // Конвертация в модели
            var employees = entries
                .Select(e => new LDAPEmployee(e))
                .ToList();

            _logger.LogInformation("Total LDAPEmployee objects created: {0}", employees.Count);
            return employees;
        }
    }
}
