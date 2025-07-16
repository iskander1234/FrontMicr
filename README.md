internal abstract class LDAPService
{
    private readonly ILogger<LDAPService> _logger;
    private readonly string _ldapServer;
    private readonly int _ldapPort;
    private readonly string _ldapBindDN;
    private readonly string _ldapPassword;
    private readonly string _ldapSearchBase;
    private readonly string _ldapCurrentFilter;
    private readonly string[] _filterKeys;

    protected LDAPService(IConfiguration configuration, ILogger<LDAPService> logger)
    {
        _logger = logger;

        var ldapSettings = configuration.GetSection("LDAPConfig");
        _ldapServer = ldapSettings["Server"] ?? throw new ArgumentNullException("LDAP Server");
        _ldapPort = int.Parse(ldapSettings["Port"]);
        _ldapBindDN = ldapSettings["BindDN"] ?? throw new ArgumentNullException("BindDN");
        _ldapPassword = ldapSettings["Password"] ?? throw new ArgumentNullException("Password");
        _ldapSearchBase = ldapSettings["SearchBase"] ?? throw new ArgumentNullException("SearchBase");

        var filterKey = $"Filters:{Filter}:Code";
        _ldapCurrentFilter = ldapSettings.GetValue<string>(filterKey) ?? throw new ArgumentNullException(filterKey);

        _filterKeys = ldapSettings.GetSection($"Filters:{Filter}:Keys").Get<string[]>() ?? Array.Empty<string>();
    }

    public virtual string Filter => "UserPerson";

    public IEnumerable<SearchResultEntry> LdapPagedSearch()
    {
        using var ldapConnection = new LdapConnection(new LdapDirectoryIdentifier(_ldapServer, _ldapPort));
        ldapConnection.Credential = new NetworkCredential(_ldapBindDN, _ldapPassword);
        ldapConnection.SessionOptions.ProtocolVersion = 3;

        _logger.LogInformation("Connecting to LDAP server {Server}:{Port}", _ldapServer, _ldapPort);

        try
        {
            ldapConnection.Bind();
        }
        catch (LdapException ex)
        {
            _logger.LogError(ex, "LDAP Bind failed: {Message}", ex.Message);
            yield break;
        }

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
            catch (Exception ex)
            {
                _logger.LogError(ex, "Search request failed: {Message}", ex.Message);
                yield break;
            }

            if (searchResponse.Controls.Length == 0 || !(searchResponse.Controls[0] is PageResultResponseControl pageResponse))
            {
                _logger.LogWarning("Server does not support paging");
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
        request.Controls.Add(new SecurityDescriptorFlagControl
        {
            SecurityMasks = SecurityMasks.Dacl | SecurityMasks.Owner
        });
        request.Controls.Add(new SearchOptionsControl(SearchOption.DomainScope));
        return request;
    }

    public void PrintAttributes(SearchResultEntry entry)
    {
        foreach (var key in _filterKeys)
        {
            if (entry.Attributes.Contains(key))
            {
                var value = entry.Attributes[key]?[0]?.ToString();
                if (!string.IsNullOrWhiteSpace(value))
                    _logger.LogInformation("{Key}: {Value}", key, value);
            }
        }
    }
}
