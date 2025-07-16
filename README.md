using System.DirectoryServices.Protocols;
using System.Net;
using Microsoft.Extensions.Configuration;

namespace DinDin.Services;

internal abstract class LDAPService
{
    protected readonly string LdapHost;
    protected readonly int LdapPort;
    protected readonly string LdapUser;
    protected readonly string LdapPassword;
    protected readonly string LdapSearchBase;

    protected virtual string Filter => "(objectClass=person)";

    protected LDAPService(IConfiguration configuration)
    {
        LdapHost = configuration["Ldap:Host"]!;
        LdapPort = int.Parse(configuration["Ldap:Port"]!);
        LdapUser = configuration["Ldap:User"]!;
        LdapPassword = configuration["Ldap:Password"]!;
        LdapSearchBase = configuration["Ldap:SearchBase"]!;
    }

    protected List<SearchResultEntry> LdapPagedSearch()
    {
        var results = new List<SearchResultEntry>();
        using var ldapConnection = new LdapConnection(new LdapDirectoryIdentifier(LdapHost, LdapPort));
        ldapConnection.AuthType = AuthType.Basic;
        ldapConnection.Bind(new NetworkCredential(LdapUser, LdapPassword));

        var pageRequestControl = new PageResultRequestControl(1000);
        var searchRequest = new SearchRequest(LdapSearchBase, Filter, SearchScope.Subtree, null);
        searchRequest.Controls.Add(pageRequestControl);

        while (true)
        {
            var searchResponse = (SearchResponse)ldapConnection.SendRequest(searchRequest);
            results.AddRange(searchResponse.Entries.Cast<SearchResultEntry>());

            var pageResponse = searchResponse.Controls.OfType<PageResultResponseControl>().FirstOrDefault();
            if (pageResponse == null || pageResponse.Cookie.Length == 0) break;

            pageRequestControl.Cookie = pageResponse.Cookie;
        }

        return results;
    }
}


 private static async Task SyncLdapAsync(IServiceProvider serviceProvider)
    {
        var configuration = serviceProvider.GetRequiredService<IConfiguration>();
        var context = serviceProvider.GetRequiredService<BpmcoreContext>();

        var ldapService = new LDAPService(configuration);
        var employees = ldapService.GetLDAPEmployees();

        var repository = new LDAPUsersRepository(context);
        await repository.UpdateByKey(employees);

        Console.WriteLine($"✅ LDAP синхронизация завершена. Обновлено: {employees.Count} записей.");
    }

