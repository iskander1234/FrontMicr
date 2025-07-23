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

    /// <summary>
    /// Полностью заменяет таблицу LdapEmployees — сначала очищает, затем вставляет все записи.
    /// </summary>
    public async Task ReplaceAllLdapEmployees(List<LDAPEmployee> employees)
    {
        // Очищаем таблицу
        await _context.Database.ExecuteSqlRawAsync("TRUNCATE TABLE \"LdapEmployees\" RESTART IDENTITY");

        // Вставляем все записи одним Bulk-вызовом
        await _context.BulkInsertAsync(employees);
    }

    /// <summary>
    /// Обновляет поле LoginAd в таблице Employees, если mail совпадает с Email в LdapEmployees
    /// </summary>
    public async Task SyncEmployeeLoginsByEmail()
    {
        var ldapList = await _context.LdapEmployees
            .Where(x => !string.IsNullOrEmpty(x.Email))
            .ToListAsync();

        var emailToLogin = ldapList
            .GroupBy(e => e.Email.ToLower())
            .ToDictionary(g => g.Key, g => g.First().Login);

        var employees = await _context.Employees
            .Where(e => !string.IsNullOrEmpty(e.Mail))
            .ToListAsync();

        foreach (var emp in employees)
        {
            var email = emp.Mail?.ToLower();
            if (email != null && emailToLogin.TryGetValue(email, out var login))
            {
                emp.LoginAd = login;
            }
        }

        await _context.SaveChangesAsync();
    }
}



public class LdapSyncService
{
    private readonly LDAPUsersRepository _ldapRepo;

    public LdapSyncService(LDAPUsersRepository ldapRepo)
    {
        _ldapRepo = ldapRepo;
    }

    public async Task SyncLdapDataAsync(List<LDAPEmployee> fetchedLdapList)
    {
        await _ldapRepo.ReplaceAllLdapEmployees(fetchedLdapList);
        await _ldapRepo.SyncEmployeeLoginsByEmail();
    }
}
