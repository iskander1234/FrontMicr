using System;
using System.Collections.Generic;
using System.DirectoryServices.Protocols;
using System.Net;
using System.Text;
using DailyDataUpdateApp.Models;
using DailyDataUpdateApp.Services;
using Microsoft.Extensions.Configuration;
//using Novell.Directory.Ldap;

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


    //public List<SearchResultEntry> LdapPagedSearch()
    //{
    //    var results = new List<SearchResultEntry>();
    //    var pageSize = 1000;
    //    var pagingControl = new PageResultRequestControl(pageSize);

    //    using var ldapConnection = new System.DirectoryServices.Protocols.LdapConnection(new LdapDirectoryIdentifier(_ldapServer + ":" + _ldapPort));
    //    ldapConnection.Credential = new NetworkCredential(_ldapBindDN, _ldapPassword);
    //    ldapConnection.Bind();

    //    while (true)
    //    {
    //        var searchRequest = new SearchRequest(_ldapSearchBase, _ldapUserPersonFilter, SearchScope.Subtree, _userFilterKeys)
    //        {
    //            Controls = { pagingControl }
    //        };

    //        var searchResponse = (SearchResponse)ldapConnection.SendRequest(searchRequest);
    //        results.AddRange(searchResponse.Entries.Cast<SearchResultEntry>());

    //        var pageResponse = (PageResultResponseControl)searchResponse.Controls[0];
    //        if (pageResponse.Cookie.Length == 0)
    //            break;

    //        pagingControl.Cookie = pageResponse.Cookie;
    //    }

    //    return results;
    //}


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

