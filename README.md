internal class LdapEmployeeSyncService
    {
        private readonly ILogger<LdapEmployeeSyncService> _logger;
        private readonly LdapDirectoryIdentifier        _identifier;
        private readonly NetworkCredential               _credentials;
        private readonly string                          _searchBase;
        private readonly string                          _filter;
        private readonly string[]                        _attributes;
        private const int PageSize = 1000;

        public LdapEmployeeSyncService(IConfiguration configuration, ILogger<LdapEmployeeSyncService> logger)
        {
            _logger    = logger;
            var server = configuration["LDAPConfig:Server"]!
                         ?? throw new ArgumentNullException("LDAPConfig:Server");
            var port   = int.TryParse(configuration["LDAPConfig:Port"], out var p) ? p : 389;
            _identifier  = new LdapDirectoryIdentifier(server, port);
            _credentials = new NetworkCredential(
                              configuration["LDAPConfig:BindDN"]!,
                              configuration["LDAPConfig:Password"]!);
            _searchBase  = configuration["LDAPConfig:SearchBase"]!;
            _filter      = configuration["LDAPConfig:Filters:UserPerson:Code"]!;
            _attributes  = configuration
                              .GetSection("LDAPConfig:Filters:UserPerson:Keys")
                              .Get<string[]>()!
                          ;
            _logger.LogInformation("LDAP Config loaded: {Server}:{Port}, BaseDN={Base}",
                server, port, _searchBase);
        }

        public List<LDAPEmployee> GetLdapEmployees()
        {
            _logger.LogInformation("Connecting to LDAP {Id}...", _identifier);
            using var connection = new LdapConnection(_identifier, _credentials, AuthType.Negotiate);
            connection.SessionOptions.ProtocolVersion = 3;
            connection.Bind();
            _logger.LogInformation("LDAP bind successful");

            var employees = new List<LDAPEmployee>();
            byte[] cookie = Array.Empty<byte>();

            do
            {
                var request = new SearchRequest(
                    _searchBase,
                    _filter,
                    SearchScope.Subtree,
                    _attributes);

                // 1) Добавляем сортировку по sAMAccountName
                var sortControl = new SortRequestControl(
                    new[] { new SortKey("sAMAccountName", null, false) });
                request.Controls.Add(sortControl);

                // 2) Добавляем пагинацию
                var pageControl = new PageResultRequestControl(PageSize) { Cookie = cookie };
                request.Controls.Add(pageControl);

                var response = (SearchResponse)connection.SendRequest(request);
                _logger.LogInformation("Received page with {Count} entries", response.Entries.Count);

                foreach (SearchResultEntry entry in response.Entries)
                {
                    try
                    {
                        employees.Add(new LDAPEmployee(entry));
                    }
                    catch (Exception ex)
                    {
                        _logger.LogWarning(ex, "Failed to parse entry {DN}", entry.DistinguishedName);
                    }
                }

                cookie = response.Controls
                    .OfType<PageResultResponseControl>()
                    .FirstOrDefault()?.Cookie
                          ?? Array.Empty<byte>();
            }
            while (cookie.Length > 0);

            _logger.LogInformation("Total records retrieved from LDAP: {Count}", employees.Count);
            return employees;
        }
    }
