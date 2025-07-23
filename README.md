public List<LDAPEmployee> GetLdapEmployees()
{
    _logger.LogInformation("Connecting to LDAP server {Server}:{Port}...", _server, _port);

    var credentials = new NetworkCredential(_bindDn, _password);
    var identifier = new LdapDirectoryIdentifier(_server, _port);

    using var connection = new LdapConnection(identifier, credentials, AuthType.Negotiate);
    connection.SessionOptions.ProtocolVersion = 3;

    try
    {
        connection.Bind();
        _logger.LogInformation("LDAP bind successful");
    }
    catch (LdapException ex)
    {
        _logger.LogError(ex, "LDAP bind failed: {Message}", ex.Message);
        throw;
    }

    _logger.LogInformation("Sending paged LDAP search request with filter: {Filter}", _filter);

    var employees = new List<LDAPEmployee>();

    var pageSize = 1000;
    var cookie = Array.Empty<byte>();
    var searchRequest = new SearchRequest(_searchBase, _filter, SearchScope.Subtree, _attributes);

    do
    {
        var pageControl = new PageResultRequestControl(pageSize) { Cookie = cookie };
        searchRequest.Controls.Clear();
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

        var pageResponse = response.Controls
            .OfType<PageResultResponseControl>()
            .FirstOrDefault();

        cookie = pageResponse?.Cookie ?? Array.Empty<byte>();

    } while (cookie != null && cookie.Length > 0);

    _logger.LogInformation("✅ Импортировано {Count} записей из LDAP.", employees.Count);
    return employees;
}
