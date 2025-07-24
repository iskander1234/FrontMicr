// File: Services/LdapEmployeeSyncService.cs
using System;
using System.Collections.Generic;
using System.DirectoryServices.Protocols;
using System.Linq;
using System.Net;
using System.Threading;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using DinDin.Models;

namespace DinDin.Services
{
    public class LdapEmployeeSyncService
    {
        private readonly ILogger<LdapEmployeeSyncService> _logger;
        private readonly LdapDirectoryIdentifier           _identifier;
        private readonly NetworkCredential                 _credentials;
        private readonly string                            _searchBase;
        private readonly string                            _filter;
        private readonly string[]                          _attributes;
        private const int PageSize       = 1000;
        private const int DelayBetweenMs = 500;   // пауза 0.5 секунды

        public LdapEmployeeSyncService(
            IConfiguration configuration,
            ILogger<LdapEmployeeSyncService> logger)
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

            _logger.LogInformation("LDAPConfig loaded: {Server}:{Port}, BaseDN={BaseDN}",
                server, port, _searchBase);
        }

        public List<LDAPEmployee> GetLdapEmployees()
        {
            _logger.LogInformation("Connecting to LDAP {Identifier}...", _identifier);
            using var connection = new LdapConnection(_identifier, _credentials, AuthType.Negotiate);
            connection.SessionOptions.ProtocolVersion = 3;
            connection.Bind();
            _logger.LogInformation("LDAP bind successful");

            var rawEntries = new List<SearchResultEntry>();
            byte[] cookie = Array.Empty<byte>();

            do
            {
                // 1) Создаем запрос
                var request = new SearchRequest(
                    _searchBase,
                    _filter,
                    SearchScope.Subtree,
                    _attributes);

                // 2) Добавляем пейджинг-контрол
                var pageControl = new PageResultRequestControl(PageSize)
                {
                    Cookie = cookie
                };
                request.Controls.Add(pageControl);

                // 3) Добавляем сортировку для консистентности результатов
                var sortControl = new SortRequestControl(new[]
                {
                    new SortKey("sAMAccountName", /* matchingRule */ null, /* reverseOrder */ false)
                });
                request.Controls.Add(sortControl);

                // 4) Отправляем запрос и получаем ответ
                var response = (SearchResponse)connection.SendRequest(request);
                _logger.LogInformation(
                    "Received page: {Count} entries; previous cookie length: {CookieLen}",
                    response.Entries.Count, cookie.Length);

                // 5) Сохраняем все записи текущей страницы
                foreach (SearchResultEntry entry in response.Entries)
                    rawEntries.Add(entry);

                // 6) Обновляем cookie для следующей страницы
                cookie = response.Controls
                            .OfType<PageResultResponseControl>()
                            .FirstOrDefault()?.Cookie
                         ?? Array.Empty<byte>();

                // 7) Пауза перед следующим запросом, если есть ещё страницы
                if (cookie.Length > 0)
                {
                    _logger.LogInformation("Sleeping {Delay}ms before next page...", DelayBetweenMs);
                    Thread.Sleep(DelayBetweenMs);
                }
            }
            while (cookie.Length > 0);

            _logger.LogInformation("Total raw entries fetched: {0}", rawEntries.Count);

            // 8) Конвертируем в модели LDAPEmployee
            var employees = rawEntries
                .Select(e => new LDAPEmployee(e))
                .ToList();

            _logger.LogInformation("Total LDAPEmployee objects created: {0}", employees.Count);
            return employees;
        }
    }
}
