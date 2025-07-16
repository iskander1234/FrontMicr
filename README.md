Microsoft.Extensions.Logging.Console

using System.DirectoryServices.Protocols;
using System.Net;
using DinDin.Models;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;

namespace DinDin.Services;

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
        var employees = new List<LDAPEmployee>();
        try
        {
            var credentials = new NetworkCredential(_bindDn, _password);
            using var connection = new LdapConnection(new LdapDirectoryIdentifier(_server, _port));
            connection.AuthType = AuthType.Basic;

            _logger.LogInformation("Connecting to LDAP server {Server}:{Port}...", _server, _port);
            connection.Bind(credentials);
            _logger.LogInformation("LDAP bind successful");

            var searchRequest = new SearchRequest(_searchBase, _filter, SearchScope.Subtree, _attributes);
            _logger.LogInformation("Sending LDAP search request with filter: {Filter}", _filter);

            var response = (SearchResponse)connection.SendRequest(searchRequest);
            _logger.LogInformation("LDAP search completed. Entries found: {Count}", response.Entries.Count);

            foreach (SearchResultEntry entry in response.Entries)
            {
                try
                {
                    var emp = new LDAPEmployee(entry);
                    employees.Add(emp);
                }
                catch (Exception ex)
                {
                    _logger.LogWarning(ex, "Failed to parse LDAP entry: {Dn}", entry.DistinguishedName);
                }
            }
        }
        catch (LdapException ex)
        {
            _logger.LogError(ex, "LDAP connection failed: {Message}", ex.Message);
            throw;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unexpected error during LDAP sync");
            throw;
        }

        return employees;
    }
}
