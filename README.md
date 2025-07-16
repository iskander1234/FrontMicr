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
