private static async Task UpdateLdapEmployees(ServiceProvider serviceProvider)
{
    // 1) Берём сервис LDAP
    var ldapService = serviceProvider.GetRequiredService<IUserLdapService>();
    var ldapEmployees = ldapService.GetLDAPEmployees();

    // 2) Берём репозиторий
    var ldapRepo = serviceProvider.GetRequiredService<LDAPUsersRepository>();

    // 3) Полностью обновляем таблицу
    await ldapRepo.UpdateByKey(ldapEmployees);
    Console.WriteLine($"🔄 Imported {ldapEmployees.Count} LDAP records into ldap_employees");

    // 4) Синхронизируем LoginAd в Employees по совпадению email
    await ldapRepo.SyncEmployeeLoginsByEmail();
    Console.WriteLine("✅ Updated LoginAd in Employees table");
}
