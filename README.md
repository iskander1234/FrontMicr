public List<LDAPEmployee> GetLdapEmployees()
{
    var results = new List<LDAPEmployee>();

    var credentials = new NetworkCredential(_bindDn, _password);
    var identifier = new LdapDirectoryIdentifier(_server, _port);

    using var connection = new LdapConnection(identifier, credentials, AuthType.Negotiate);
    connection.SessionOptions.ProtocolVersion = 3;
    connection.Bind();

    var pageSize = 1000;
    var cookie = Array.Empty<byte>();

    do
    {
        var request = new SearchRequest(
            _searchBase,
            _filter,
            SearchScope.Subtree,
            _attributes
        );

        request.Controls.Add(new PageResultRequestControl(pageSize) { Cookie = cookie });
        var response = (SearchResponse)connection.SendRequest(request);

        foreach (SearchResultEntry entry in response.Entries)
        {
            try
            {
                var emp = new LDAPEmployee(entry);
                results.Add(emp);
            }
            catch (Exception ex)
            {
                _logger.LogWarning(ex, "⛔ Ошибка при парсинге LDAP записи");
            }
        }

        var pageResponse = response.Controls.OfType<PageResultResponseControl>().FirstOrDefault();
        cookie = pageResponse?.Cookie ?? Array.Empty<byte>();

    } while (cookie.Length > 0);

    _logger.LogInformation("✅ Всего записей: {Count}", results.Count);
    return results;
}
