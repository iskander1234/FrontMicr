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
        private readonly string _ldapPath;
        private readonly string _bindDn;
        private readonly string _password;
        private readonly string _filter;
        private readonly string[] _attributes;

        public LdapEmployeeSyncService(IConfiguration configuration, ILogger<LdapEmployeeSyncService> logger)
        {
            _logger = logger;
            var server = configuration["LDAPConfig:Server"] ?? throw new ArgumentNullException("LDAPConfig:Server is required");
            var port = configuration["LDAPConfig:Port"] ?? "389";
            _ldapPath = $"LDAP://{server}:{port}/{configuration["LDAPConfig:SearchBase"]}";
            _bindDn = configuration["LDAPConfig:BindDN"] ?? throw new ArgumentNullException("LDAPConfig:BindDN is required");
            _password = configuration["LDAPConfig:Password"] ?? throw new ArgumentNullException("LDAPConfig:Password is required");
            _filter = configuration["LDAPConfig:Filters:UserPerson:Code"] ?? "(objectClass=user)";
            _attributes = configuration.GetSection("LDAPConfig:Filters:UserPerson:Keys").Get<string[]>() ?? Array.Empty<string>();

            _logger.LogInformation("DirectorySearcher configured: {Path}", _ldapPath);
        }

        public List<LDAPEmployee> GetLdapEmployees()
        {
            var employees = new List<LDAPEmployee>();
            try
            {
                using var entry = new DirectoryEntry(_ldapPath, _bindDn, _password, AuthenticationTypes.Secure);
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

                int idx = 0;
                foreach (SearchResult res in results)
                {
                    idx++;
                    try
                    {
                        var emp = new LDAPEmployee(res);
                        employees.Add(emp);
                        if (idx <= 5)
                            _logger.LogInformation("Parsed #{Idx}: Login={Login}, Name={Name}", idx, emp.Login, emp.Name);
                    }
                    catch (Exception ex)
                    {
                        _logger.LogWarning(ex, "Failed to parse entry #{Idx}, Path={Path}", idx, res.Path);
                    }
                }

                _logger.LogInformation("✅ Completed parsing entries. Total added: {Count}", employees.Count);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "DirectorySearcher failed: {Message}", ex.Message);
                throw;
            }

            return employees;
        }
    }
}
