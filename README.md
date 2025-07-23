using DinDin.Models;
using EFCore.BulkExtensions;
using Microsoft.EntityFrameworkCore;

namespace DinDin.Repositories;

public class LDAPUsersRepository
{
    private readonly BpmcoreContext _context;

    public LDAPUsersRepository(BpmcoreContext context)
    {
        _context = context;
    }

    public async Task UpdateByKey(List<LDAPEmployee> employees)
    {
        var logins = employees.Select(e => e.Login).ToList();
        var existing = await _context.LdapEmployees
            .Where(x => logins.Contains(x.Login))
            .ToListAsync();

        var toUpdate = employees.Where(e => existing.Any(x => x.Login == e.Login)).ToList();
        var toInsert = employees.Where(e => existing.All(x => x.Login != e.Login)).ToList();

        foreach (var updated in toUpdate)
        {
            var record = existing.First(x => x.Login == updated.Login);
            _context.Entry(record).CurrentValues.SetValues(updated);
        }

        _context.LdapEmployees.AddRange(toInsert);
        await _context.SaveChangesAsync();
    }
}
