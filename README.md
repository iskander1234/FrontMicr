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
            _server = configuration["LDAPConfig:Server"] ?? throw new ArgumentNullException(nameof(configuration), "LDAPConfig:Server is required");
            _port = int.TryParse(configuration["LDAPConfig:Port"], out var p) ? p : 389;
            _bindDn = configuration["LDAPConfig:BindDN"] ?? throw new ArgumentNullException(nameof(configuration), "LDAPConfig:BindDN is required");
            _password = configuration["LDAPConfig:Password"] ?? throw new ArgumentNullException(nameof(configuration), "LDAPConfig:Password is required");
            _searchBase = configuration["LDAPConfig:SearchBase"] ?? throw new ArgumentNullException(nameof(configuration), "LDAPConfig:SearchBase is required");

            _filter = configuration["LDAPConfig:Filters:UserPerson:Code"] ?? "(objectClass=user)";
            _attributes = configuration.GetSection("LDAPConfig:Filters:UserPerson:Keys").Get<string[]>()
                          ?? Array.Empty<string>();

            _logger.LogInformation("LDAP Config loaded: Server={Server}, Port={Port}, BindDN={BindDN}, SearchBase={SearchBase}",
                _server, _port, _bindDn, _searchBase);
        }

        public List<LDAPEmployee> GetLdapEmployees()
        {
            _logger.LogInformation("Connecting to LDAP via DirectorySearcher: {Server}:{Port}...", _server, _port);
            var employees = new List<LDAPEmployee>();

            try
            {
                // Подключение через System.DirectoryServices
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

                // Загрузка атрибутов
                foreach (var attr in _attributes)
                    searcher.PropertiesToLoad.Add(attr);

                _logger.LogInformation("Starting DirectorySearcher.FindAll()...");
                using var results = searcher.FindAll();
                _logger.LogInformation("DirectorySearcher returned {Count} entries", results.Count);

                // Преобразование результатов в модели
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




















    [Table("ldap_employees")]
    public class LDAPEmployee
    {
        [Key]
        [Column("id")]
        public int Id { get; set; }

        [Column("user_status_code")]
        public int UserStatusCode { get; set; }

        [Column("is_disabled")]
        public bool IsDisabled { get; set; }

        [Column("given_name")]
        public string? GivenName { get; set; }

        [Column("name")]
        public string Name { get; set; } = string.Empty;

        [Column("position_name")]
        public string? PositionName { get; set; }

        [Column("title")]
        public string? Title { get; set; }

        [Column("department")]
        public string? Department { get; set; }

        [Column("login")]
        public string Login { get; set; } = string.Empty;

        [Column("local_phone")]
        public string? LocalPhone { get; set; }

        [Column("mobile_number")]
        public string? MobileNumber { get; set; }

        [Column("city")]
        public string? City { get; set; }

        [Column("location_address")]
        public string? LocationAddress { get; set; }

        [Column("postal_code")]
        public string? PostalCode { get; set; }

        [Column("company")]
        public string? Company { get; set; }

        [Column("manager")]
        public string? Manager { get; set; }

        [Column("email")]
        public string? Email { get; set; }

        [Column("member_of")]
        public string[]? MemberOf { get; set; }

        [Column("member_of_string", TypeName = "varchar")]
        public string? MemberOfString { get; set; }

        [Column("department_id")]
        public int? DepartmentId { get; set; }

        [Column("department_details")]
        public JsonDocument? DepartmentDetails { get; set; }

        [Column("group_id")]
        public int? GroupId { get; set; }

        [Column("is_manager")]
        public bool IsManager { get; set; }

        public LDAPEmployee() { }

        // Constructor for LdapConnection (SearchResultEntry)
        public LDAPEmployee(SearchResultEntry entry)
        {
            UserStatusCode = entry.GetInt("userAccountControl");
            IsDisabled = (UserStatusCode & 2) != 0;
            GivenName = entry.GetString("givenName");
            Name = entry.GetString("cn") ?? string.Empty;
            PositionName = entry.GetString("description");
            Title = entry.GetString("title");
            Department = entry.GetString("department");
            Login = entry.GetString("sAMAccountName") ?? string.Empty;
            LocalPhone = entry.GetString("telephoneNumber");
            MobileNumber = entry.GetString("mobile");
            City = entry.GetString("l");
            LocationAddress = entry.GetString("streetAddress");
            PostalCode = entry.GetString("postalCode");
            Company = entry.GetString("company");
            Manager = entry.GetString("manager");
            Email = entry.GetString("mail");
            MemberOf = entry.GetStringArray("memberOf");
            MemberOfString = MemberOf == null ? null : string.Join(";", MemberOf);
            IsManager = false;
        }

        // New constructor for DirectorySearcher (SearchResult)
        public LDAPEmployee(SearchResult entry)
        {
            UserStatusCode = GetInt(entry, "userAccountControl");
            IsDisabled = (UserStatusCode & 2) != 0;
            GivenName = GetString(entry, "givenName");
            Name = GetString(entry, "cn") ?? string.Empty;
            PositionName = GetString(entry, "description");
            Title = GetString(entry, "title");
            Department = GetString(entry, "department");
            Login = GetString(entry, "sAMAccountName") ?? string.Empty;
            LocalPhone = GetString(entry, "telephoneNumber");
            MobileNumber = GetString(entry, "mobile");
            City = GetString(entry, "l");
            LocationAddress = GetString(entry, "streetAddress");
            PostalCode = GetString(entry, "postalCode");
            Company = GetString(entry, "company");
            Manager = GetString(entry, "manager");
            Email = GetString(entry, "mail");
            if (entry.Properties.Contains("memberOf"))
            {
                MemberOf = entry.Properties["memberOf"].Cast<object>()
                                .Select(o => o.ToString()!)
                                .ToArray();
                MemberOfString = string.Join(";", MemberOf);
            }
            else
            {
                MemberOf = Array.Empty<string>();
                MemberOfString = null;
            }
            IsManager = false;
        }

        private static string? GetString(SearchResult entry, string propName)
        {
            if (entry.Properties.Contains(propName) && entry.Properties[propName].Count > 0)
                return entry.Properties[propName][0]?.ToString();
            return null;
        }

        private static int GetInt(SearchResult entry, string propName)
        {
            var s = GetString(entry, propName);
            return int.TryParse(s, out var i) ? i : 0;
        }
    }
}

