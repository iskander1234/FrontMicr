"LDAPConfig": {
  "Server": "ldap://yourdomain.kz",     // или просто yourdomain.kz
  "Port": "389",
  "BindDN": "user@yourdomain.kz",       // либо CN=Имя,OU=Users,DC=yourdomain,DC=kz
  "Password": "парольПользователя",
  "SearchBase": "DC=yourdomain,DC=kz",
  "Filters": {
    "UserPerson": {
      "Code": "(&(objectClass=user)(objectCategory=PERSON))",
      "Keys": [
        "givenName",
        "sAMAccountName",
        "department",
        "member",
        "memberof",
        "manager",
        "cn",
        "UserAccountControl&2",
        "UserAccountControl",
        "mail",
        "sn",
        "wWWWHomePage",
        "homePhone",
        "streetAddress",
        "postOfficeBox",
        "l",
        "st",
        "postalCode",
        "c",
        "company",
        "title",
        "telephoneNumber",
        "facsimileTelephoneNumber",
        "description",
        "ipPhone"
      ]
    }
  }
}


using Microsoft.Extensions.Configuration;
using System.DirectoryServices.Protocols;
using System.Net;

namespace DinDin.Services;

public class LdapEmployeeSyncService
{
    private readonly string _server;
    private readonly int _port;
    private readonly string _bindDn;
    private readonly string _password;
    private readonly string _searchBase;
    private readonly string _userFilter;

    public LdapEmployeeSyncService(IConfiguration configuration)
    {
        _server = configuration["LDAPConfig:Server"]!;
        _port = int.Parse(configuration["LDAPConfig:Port"]!);
        _bindDn = configuration["LDAPConfig:BindDN"]!;
        _password = configuration["LDAPConfig:Password"]!;
        _searchBase = configuration["LDAPConfig:SearchBase"]!;
        _userFilter = configuration["LDAPConfig:Filters:UserPerson:Code"]!;
    }

    public List<SearchResultEntry> GetLdapEntries()
    {
        var result = new List<SearchResultEntry>();

        using var ldapConnection = new LdapConnection(new LdapDirectoryIdentifier(_server, _port));
        ldapConnection.AuthType = AuthType.Basic;
        ldapConnection.Bind(new NetworkCredential(_bindDn, _password));

        var request = new SearchRequest(_searchBase, _userFilter, SearchScope.Subtree, null);

        var pageRequest = new PageResultRequestControl(500);
        request.Controls.Add(pageRequest);

        while (true)
        {
            var response = (SearchResponse)ldapConnection.SendRequest(request);
            result.AddRange(response.Entries.Cast<SearchResultEntry>());

            var pageResponse = response.Controls.OfType<PageResultResponseControl>().FirstOrDefault();
            if (pageResponse == null || pageResponse.Cookie.Length == 0)
                break;

            pageRequest.Cookie = pageResponse.Cookie;
        }

        return result;
    }
}
