public List<LDAPEmployee> GetLdapEmployees()
{
    _logger.LogInformation("Connecting to LDAP server {Server}:{Port}...", _server, _port);

    var credentials = new NetworkCredential(_bindDn, _password);
    var identifier = new LdapDirectoryIdentifier(_server, _port);

    using var connection = new LdapConnection(identifier, credentials, AuthType.Negotiate);
    connection.SessionOptions.ProtocolVersion = 3;

    connection.Bind();

    var employees = new List<LDAPEmployee>();
    var cookie = Array.Empty<byte>();
    var pageSize = 1000;

    do
    {
        var searchRequest = new SearchRequest(
            _searchBase,
            _filter,
            SearchScope.Subtree,
            _attributes
        );

        var pageControl = new PageResultRequestControl(pageSize)
        {
            Cookie = cookie
        };
        searchRequest.Controls.Add(pageControl);

        var response = (SearchResponse)connection.SendRequest(searchRequest);

        foreach (SearchResultEntry entry in response.Entries)
        {
            try
            {
                var emp = new LDAPEmployee(entry);
                employees.Add(emp);
            }
            catch (Exception ex)
            {
                _logger.LogWarning(ex, "Failed to parse entry");
            }
        }

        var responseControl = response.Controls
            .OfType<PageResultResponseControl>()
            .FirstOrDefault();

        cookie = responseControl?.Cookie ?? Array.Empty<byte>();
    }
    while (cookie != null && cookie.Length > 0);

    _logger.LogInformation("✅ Импортировано {Count} записей из LDAP.", employees.Count);
    return employees;
}
