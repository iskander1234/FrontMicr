using System.DirectoryServices.Protocols;
using Microsoft.Extensions.Configuration;
using DinDin.Models;

namespace DinDin.Services;

internal class UserLdapService : LDAPService
{
    public UserLdapService(IConfiguration configuration) : base(configuration) { }

    public List<LDAPEmployee> GetLDAPEmployees()
    {
        var entries = LdapPagedSearch();

        var employees = new List<LDAPEmployee>();
        foreach (var entry in entries)
        {
            var employee = new LDAPEmployee
            {
                GivenName = entry.Attributes["givenName"]?[0]?.ToString(),
                Name = entry.Attributes["cn"]?[0]?.ToString(),
                Login = entry.Attributes["sAMAccountName"]?[0]?.ToString(),
                Mail = entry.Attributes["mail"]?[0]?.ToString(),
                UserStatusCode = int.TryParse(entry.Attributes["userAccountControl"]?[0]?.ToString(), out var code) ? code : 0,
                IsDisabled = false
            };

            employees.Add(employee);
        }

        return employees;
    }
}
