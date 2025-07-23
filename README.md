private static async Task SyncLdap(IServiceProvider provider)
    {
        var context = provider.GetRequiredService<BpmcoreContext>();
        var ldapService = provider.GetRequiredService<LdapEmployeeSyncService>();
        var ldapUsersRepo = provider.GetRequiredService<LDAPUsersRepository>();

        var ldapEmployees = ldapService.GetLdapEmployees();

        await ldapUsersRepo.UpdateByKey(ldapEmployees);

        Console.WriteLine($"✅ Импортировано {ldapEmployees.Count} записей в таблицу ldap_employees.");
    }
