using DinDin.Models;
using EFCore.BulkExtensions;
using Microsoft.EntityFrameworkCore;

namespace DinDin.Repositories
{
    public class LDAPUsersRepository
    {
        private readonly BpmcoreContext _ctx;
        public LDAPUsersRepository(BpmcoreContext ctx) => _ctx = ctx;

        /// <summary>
        /// Полностью заменяет таблицу ldap_employees:
        /// TRUNCATE + BulkInsert.
        /// </summary>
        public async Task UpdateByKey(List<LDAPEmployee> updatedLdapEmployees)
        {
            await _ctx.BulkInsertOrUpdateAsync(updatedLdapEmployees.ToList(), options =>
            {
                options.BatchSize = 1000;
                options.IncludeGraph = false;
                options.UpdateByProperties = ["Login"];
                options.SetOutputIdentity = true;
                options.UseTempDB = false;
                options.PropertiesToExclude =
                [
                    "Id", "GroupId", "DepartmentId",
                    //"DepartmentDetails",
                    "IsManager"
                ];
            });
        }
        /// <summary>
        /// Обновляет LoginAd в основной таблице Employees по совпадению email.
        /// </summary>
        public async Task SyncEmployeeLoginsByEmail()
        {
            var ldapList = await _ctx.LdapEmployees
                .Where(x => !string.IsNullOrEmpty(x.Email))
                .ToListAsync();

            var map = ldapList
                .GroupBy(e => e.Email!.ToLower())
                .ToDictionary(g => g.Key, g => g.First().Login);

            var employees = await _ctx.Employees
                .Where(e => !string.IsNullOrEmpty(e.Mail))
                .ToListAsync();

            foreach (var emp in employees)
            {
                var mail = emp.Mail!.ToLower();
                if (map.TryGetValue(mail, out var login))
                    emp.LoginAd = login;
            }

            await _ctx.SaveChangesAsync();
        }
    }
}

using System.DirectoryServices.Protocols;
using System.Net;
using Microsoft.Extensions.Configuration;

namespace DinDin.Services;

internal abstract class LDAPService
{
    private const int PageSize = 1000;
    private readonly string _ldapServer;
    private readonly int _ldapPort;
    private readonly string _ldapBindDN;
    private readonly string _ldapPassword;
    private readonly string _ldapSearchBase;
    private readonly string _ldapCurrentFilter;
    private readonly string[] _filterKeys;

    public virtual string Filter => "UserPerson";

    protected LDAPService(IConfiguration configuration)
    {
        var ldapSettings = configuration.GetSection("LDAPConfig");
        _ldapServer = ldapSettings["Server"] ?? throw new NullReferenceException("LDAP Server is not specified");
        _ldapPort = int.Parse(ldapSettings["Port"]);
        _ldapBindDN = ldapSettings["BindDN"] ?? throw new NullReferenceException("BindDN is not specified");
        _ldapPassword = ldapSettings["Password"] ?? throw new NullReferenceException("Password is not specified");
        _ldapSearchBase = ldapSettings["SearchBase"] ?? throw new NullReferenceException("SearchBase is not specified");

        _ldapCurrentFilter = ldapSettings.GetSection($"Filters:{Filter}:Code").Value ?? throw new NullReferenceException(Filter + ":Code filter is not specified");

        _filterKeys = ldapSettings.GetSection($"Filters:{Filter}:Keys").GetChildren()?.Select(e => e.Value)?.ToArray() ?? throw new NullReferenceException(Filter + " Keys filter is not specified"); ;
    }

    public IEnumerable<SearchResultEntry> LdapPagedSearch()
    {
        using var ldapConnection = new LdapConnection(new LdapDirectoryIdentifier(_ldapServer + ":" + _ldapPort));
        ldapConnection.Credential = new NetworkCredential(_ldapBindDN, _ldapPassword);
        ldapConnection.SessionOptions.ProtocolVersion = 3;
        ldapConnection.Bind();

        var searchRequest = GetRequest(_ldapSearchBase, _ldapCurrentFilter, _filterKeys, SearchScope.Subtree);
        var pageRequest = new PageResultRequestControl(PageSize);
        searchRequest.Controls.Add(pageRequest);

        while (true)
        {
            SearchResponse searchResponse;
            try
            {
                searchResponse = (SearchResponse)ldapConnection.SendRequest(searchRequest);
            }
            catch (Exception e)
            {
                Console.WriteLine(_ldapCurrentFilter);
                Console.WriteLine("\nUnexpected exception occurred:\n\t{0}: {1}", e.GetType().Name, e.Message);
                yield break;
            }

            if (searchResponse.Controls.Length != 1 || !(searchResponse.Controls[0] is PageResultResponseControl pageResponse))
            {
                Console.WriteLine("Server does not support paging");
                yield break;
            }

            foreach (SearchResultEntry entry in searchResponse.Entries)
            {
                yield return entry;
            }

            if (pageResponse.Cookie.Length == 0)
                break;

            pageRequest.Cookie = pageResponse.Cookie;
        }
    }


    private static SearchRequest GetRequest(string dn, string filter, string[] returnAttrs, SearchScope scope = SearchScope.Subtree)
    {
        var request = new SearchRequest(dn, filter, scope, returnAttrs);

        // Set controls for security descriptor and domain scope
        var searchControl = new SearchOptionsControl(System.DirectoryServices.Protocols.SearchOption.DomainScope);
        var securityDescriptorFlagControl = new SecurityDescriptorFlagControl
        {
            SecurityMasks = SecurityMasks.Dacl | SecurityMasks.Owner
        };
        request.Controls.Add(securityDescriptorFlagControl);
        request.Controls.Add(searchControl);

        return request;
    }

    public void PrintAttributes(in SearchResultEntry entry)
    {
        foreach (var key in _filterKeys)
        {
            var selectedEntry = entry.Attributes[key];
            if (selectedEntry != null)
            {
                var value = selectedEntry != null ? selectedEntry[0].ToString() : null;

                if (value != null)
                    Console.WriteLine($"{key}: {value}");
            }
        }
        Console.WriteLine();
    }
}
using DinDin.Interface;
using DinDin.Models;
using Microsoft.Extensions.Configuration;

namespace DinDin.Services;

internal class UserLDAPService(IConfiguration configuration) : LDAPService(configuration), IUserLdapService
{
    public override string Filter => "UserPerson";

    public List<LDAPEmployee> GetLDAPEmployees()
    {
        var results = LdapPagedSearch();
        var employeeResults = results
            .Select(e => new LDAPEmployee(e))
            .ToList();
        return employeeResults;
    }
}
using System.DirectoryServices.Protocols;
using DinDin.Models;

namespace DinDin.Interface;

internal interface IUserLdapService
{
    public void PrintAttributes(in SearchResultEntry entry);
    public List<LDAPEmployee> GetLDAPEmployees();
}





