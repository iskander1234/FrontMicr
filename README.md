using System.DirectoryServices.Protocols;
using System.Net;
using DinDin.Models;
using Microsoft.Extensions.Configuration;

namespace DinDin.Services;

public class LdapEmployeeSyncService
{
    private readonly string _server;
    private readonly int _port;
    private readonly string _bindDn;
    private readonly string _password;
    private readonly string _searchBase;
    private readonly string _filter;
    private readonly string[] _attributes;

    public LdapEmployeeSyncService(IConfiguration configuration)
    {
        _server = configuration["LDAPConfig:Server"] ?? throw new ArgumentNullException("Server");
        _port = int.TryParse(configuration["LDAPConfig:Port"], out var port) ? port : 389;
        _bindDn = configuration["LDAPConfig:BindDN"] ?? throw new ArgumentNullException("BindDN");
        _password = configuration["LDAPConfig:Password"] ?? throw new ArgumentNullException("Password");
        _searchBase = configuration["LDAPConfig:SearchBase"] ?? throw new ArgumentNullException("SearchBase");

        _filter = configuration["LDAPConfig:Filters:UserPerson:Code"] ?? "(objectClass=user)";
        _attributes = configuration.GetSection("LDAPConfig:Filters:UserPerson:Keys").Get<string[]>() ?? Array.Empty<string>();
        
    }

    public List<LDAPEmployee> GetLdapEmployees()
    {
        var credentials = new NetworkCredential(_bindDn, _password);
        using var connection = new LdapConnection(new LdapDirectoryIdentifier(_server, _port));
        connection.AuthType = AuthType.Basic;
        connection.Bind(credentials);

        var searchRequest = new SearchRequest(_searchBase, _filter, SearchScope.Subtree, _attributes);
        var response = (SearchResponse)connection.SendRequest(searchRequest);

        var employees = new List<LDAPEmployee>();

        foreach (SearchResultEntry entry in response.Entries)
        {
            try
            {
                var emp = new LDAPEmployee(entry);
                employees.Add(emp);
            }
            catch
            {
                // log or ignore
            }
        }

        return employees;
    }
}
