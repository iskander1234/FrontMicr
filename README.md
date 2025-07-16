static async Task SyncLdap(IServiceProvider provider)
{
    var context = provider.GetRequiredService<BpmcoreContext>();
    var config = provider.GetRequiredService<IConfiguration>();

    var repo = new LDAPUsersRepository(context);
    var service = new LdapEmployeeSyncService(config);

    var entries = service.GetLdapEntries();
    var employees = entries.Select(x => new LDAPEmployee(x)).ToList();

    await repo.UpdateByKey(employees);
    Console.WriteLine($"✅ Импортировано: {employees.Count} записей в таблицу ldap_employees");
}
