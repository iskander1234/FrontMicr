public List<LDAPEmployee> GetLdapEmployees()
{
    var result = new List<LDAPEmployee>();

    using var ldapConnection = new LdapConnection(new LdapDirectoryIdentifier(_server, _port));
    var credentials = new NetworkCredential(_username, _password);
    ldapConnection.Credential = credentials;
    ldapConnection.AuthType = AuthType.Negotiate;
    ldapConnection.Bind();

    var pageSize = 1000;
    var pageRequestControl = new PageResultRequestControl(pageSize);
    var searchRequest = new SearchRequest(_searchBase, "(&(objectClass=user)(objectCategory=person))", SearchScope.Subtree, null);

    while (true)
    {
        searchRequest.Controls.Clear();
        searchRequest.Controls.Add(pageRequestControl);

        var response = (SearchResponse)ldapConnection.SendRequest(searchRequest);

        foreach (SearchResultEntry entry in response.Entries)
        {
            // Преобразование entry в LDAPEmployee (твой метод)
            var employee = ConvertToLdapEmployee(entry);
            result.Add(employee);
        }

        var pageResponse = response.Controls.OfType<PageResultResponseControl>().FirstOrDefault();
        if (pageResponse == null || pageResponse.Cookie.Length == 0)
            break;

        pageRequestControl.Cookie = pageResponse.Cookie;
    }

    return result;
}
