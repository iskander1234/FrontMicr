using System;
using System.Collections;
using System.Collections.Generic;
using System.DirectoryServices;
using System.Linq;
using DinDin.Models;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;

namespace DinDin.Services
{
    public class LdapEmployeeSyncService
    {
        private readonly ILogger<LdapEmployeeSyncService> _logger;
        private readonly string _ldapPath;
        private readonly string _bindDn;
        private readonly string _password;
        private readonly string _filter;
        private readonly string[] _properties;

        public LdapEmployeeSyncService(IConfiguration config, ILogger<LdapEmployeeSyncService> logger)
        {
            _logger = logger;
            var server = config["LDAPConfig:Server"] ?? throw new ArgumentNullException("Server");
            var port = config["LDAPConfig:Port"] ?? "389";
            var baseDn = config["LDAPConfig:SearchBase"] ?? throw new ArgumentNullException("SearchBase");
            _ldapPath = $"LDAP://{server}:{port}/{baseDn}";
            _bindDn = config["LDAPConfig:BindDN"] ?? throw new ArgumentNullException("BindDN");
            _password = config["LDAPConfig:Password"] ?? throw new ArgumentNullException("Password");
            _filter = config["LDAPConfig:Filters:UserPerson:Code"] ?? "(objectClass=user)";
            _properties = config.GetSection("LDAPConfig:Filters:UserPerson:Keys").Get<string[]>()
                          ?? Array.Empty<string>();

            _logger.LogInformation("LDAP DirectorySearcher configured: {Path}", _ldapPath);
        }

        public List<LDAPEmployee> GetLdapEmployees()
        {
            var employees = new List<LDAPEmployee>();
            try
            {
                using var entry = new DirectoryEntry(_ldapPath, _bindDn, _password, AuthenticationTypes.Secure);
                using var searcher = new DirectorySearcher(entry, _filter, _properties)
                {
                    PageSize = 500,
                    ReferralChasing = ReferralChasingOption.All,
                    SearchScope = SearchScope.Subtree,
                    CacheResults = false
                };

                _logger.LogInformation("Starting search via DirectorySearcher...");
                using var results = searcher.FindAll();
                _logger.LogInformation("DirectorySearcher returned {Count} results", results.Count);

                foreach (SearchResult res in results)
                {
                    try
                    {
                        var emp = new LDAPEmployee
                        {
                            UserStatusCode = GetInt(res, "userAccountControl"),
                            IsDisabled = (GetInt(res, "userAccountControl") & 2) != 0,
                            GivenName = GetString(res, "givenName"),
                            Name = GetString(res, "cn") ?? string.Empty,
                            PositionName = GetString(res, "position"),
                            Title = GetString(res, "title"),
                            Department = GetString(res, "department"),
                            Login = GetString(res, "sAMAccountName") ?? string.Empty,
                            LocalPhone = GetString(res, "telephoneNumber"),
                            MobileNumber = GetString(res, "mobile"),
                            City = GetString(res, "l"),
                            LocationAddress = GetString(res, "streetAddress"),
                            PostalCode = GetString(res, "postalCode"),
                            Company = GetString(res, "company"),
                            Manager = GetString(res, "manager"),
                            Email = GetString(res, "mail"),
                            MemberOf = GetArray(res, "memberOf")
                        };
                        emp.MemberOfString = emp.MemberOf == null ? null : string.Join(";", emp.MemberOf);
                        employees.Add(emp);
                    }
                    catch (Exception ex)
                    {
                        _logger.LogWarning(ex, "Error parsing entry {Path}", res.Path);
                    }
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "DirectorySearcher failed: {Message}", ex.Message);
                throw;
            }

            _logger.LogInformation("🎯 Total records via DirectorySearcher: {Count}", employees.Count);
            return employees;
        }

        private static string? GetString(SearchResult res, string propName) =>
            res.Properties.Contains(propName) && res.Properties[propName].Count > 0
                ? res.Properties[propName][0]?.ToString()
                : null;

        private static int GetInt(SearchResult res, string propName) =>
            int.TryParse(GetString(res, propName), out var i) ? i : 0;

        private static string[]? GetArray(SearchResult res, string propName) =>
            res.Properties.Contains(propName)
                ? res.Properties[propName]
                    .Cast<object>()
                    .Select(o => o.ToString()!)
                    .ToArray()
                : null;
    }
}
