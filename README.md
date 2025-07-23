private static async Task SyncLdap(IServiceProvider provider)
{
    var context = provider.GetRequiredService<BpmcoreContext>();
    var ldapService = provider.GetRequiredService<LdapEmployeeSyncService>();
    var ldapUsersRepo = provider.GetRequiredService<LDAPUsersRepository>();

    // Получаем всех LDAP сотрудников (без ограничения 1000)
    var ldapEmployees = ldapService.GetLdapEmployees();

    // Полная замена таблицы LdapEmployees
    await ldapUsersRepo.ReplaceAllLdapEmployees(ldapEmployees);

    Console.WriteLine($"✅ Импортировано {ldapEmployees.Count} записей в таблицу ldap_employees.");

    // Обновляем LoginAd в таблице Employees по email
    await ldapUsersRepo.SyncEmployeeLoginsByEmail();
    Console.WriteLine($"🔄 Обновлены поля LoginAd в таблице employees.");
}
