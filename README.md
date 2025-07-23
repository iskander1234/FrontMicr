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

    var employees = new List<LDAPEmployee>();
    var pageSize = 1000;
    var cookie = Array.Empty<byte>();

    do
    {
        // Создаём новый запрос на каждой итерации
        var searchRequest = new SearchRequest(_searchBase, _filter, SearchScope.Subtree, _attributes);

        // Добавляем постраничную пагинацию
        var pageControl = new PageResultRequestControl(pageSize) { Cookie = cookie };
        searchRequest.Controls.Add(pageControl);

        // Выполняем запрос
        var response = (SearchResponse)connection.SendRequest(searchRequest);

        // Обрабатываем записи
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

        // Извлекаем cookie для следующей страницы
        var pageResponse = response.Controls.OfType<PageResultResponseControl>().FirstOrDefault();
        cookie = pageResponse?.Cookie ?? Array.Empty<byte>();

        _logger.LogInformation("Загружено ещё {Count} записей, всего: {Total}", response.Entries.Count, employees.Count);

    } while (cookie != null && cookie.Length > 0);

    _logger.LogInformation("✅ Всего импортировано из LDAP: {Count} записей", employees.Count);
    return employees;
}
