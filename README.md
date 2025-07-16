static async Task SyncLdap(IServiceProvider provider)
    {
        var context = provider.GetRequiredService<BpmcoreContext>();
        var repo = new LDAPUsersRepository(context);
        var service = provider.GetRequiredService<LdapEmployeeSyncService>(); // ВАЖНО: теперь получаем из DI
        var entries = service.GetLdapEmployees();
        var employees = entries.Select(e => new LDAPEmployee(e)).ToList();

        await repo.UpdateByKey(employees);
        Console.WriteLine($"✅ Импортировано: {employees.Count} записей в таблицу ldap_employees");
    }
