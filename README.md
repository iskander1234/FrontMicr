using System;
using System.Collections.Generic;
using System.DirectoryServices;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using DinDin.Models;

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
            _server = configuration["LDAPConfig:Server"] ?? throw new ArgumentNullException("LDAPConfig:Server is required");
            _port = int.TryParse(configuration["LDAPConfig:Port"], out var port) ? port : 389;
            _bindDn = configuration["LDAPConfig:BindDN"] ?? throw new ArgumentNullException("LDAPConfig:BindDN is required");
            _password = configuration["LDAPConfig:Password"] ?? throw new ArgumentNullException("LDAPConfig:Password is required");
            _searchBase = configuration["LDAPConfig:SearchBase"] ?? throw new ArgumentNullException("LDAPConfig:SearchBase is required");
            _filter = configuration["LDAPConfig:Filters:UserPerson:Code"] ?? "(objectClass=user)";
            _attributes = configuration.GetSection("LDAPConfig:Filters:UserPerson:Keys").Get<string[]>() ?? Array.Empty<string>();

            _logger.LogInformation("LDAP Config loaded: Server={Server}, Port={Port}, BindDN={BindDN}, SearchBase={SearchBase}",
                _server, _port, _bindDn, _searchBase);
        }

        public List<LDAPEmployee> GetLdapEmployees()
        {
            _logger.LogInformation("Connecting to LDAP via DirectorySearcher: {Server}:{Port}...", _server, _port);
            var employees = new List<LDAPEmployee>();

            try
            {
                using var entry = new DirectoryEntry(
                    $"LDAP://{_server}:{_port}/{_searchBase}",
                    _bindDn,
                    _password,
                    AuthenticationTypes.Secure);

                using var searcher = new DirectorySearcher(entry)
                {
                    Filter = _filter,
                    PageSize = 1000,
                    ReferralChasing = ReferralChasingOption.All,
                    SearchScope = SearchScope.Subtree,
                    CacheResults = false
                };

                foreach (var attr in _attributes)
                    searcher.PropertiesToLoad.Add(attr);

                _logger.LogInformation("Starting DirectorySearcher.FindAll()...");
                using var results = searcher.FindAll();
                _logger.LogInformation("DirectorySearcher returned {Count} entries", results.Count);

                foreach (SearchResult res in results)
                {
                    try
                    {
                        employees.Add(new LDAPEmployee(res));
                    }
                    catch (Exception ex)
                    {
                        _logger.LogWarning(ex, "Failed to parse entry {Path}", res.Path);
                    }
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "DirectorySearcher failed: {Message}", ex.Message);
                throw;
            }

            _logger.LogInformation("🎯 Total records retrieved from LDAP: {Count}", employees.Count);
            return employees;
        }
    }
}
