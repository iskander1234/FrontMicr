private static ServiceProvider ConfigureServices(IConfiguration configuration)
{
    var connectionString = configuration.GetConnectionString("BpmCore") ?? throw new Exception("ConnectionString BpmCore was not found");

    return new ServiceCollection()
        .AddSingleton<IConfiguration>(configuration)
        .AddDbContext<BpmcoreContext>(options => options.UseNpgsql(connectionString))
        .AddSingleton<OneCService>()
        .AddSingleton<DepartmentRepository>()
        .AddSingleton<EmployeeRepository>()
        .AddSingleton<LdapEmployeeSyncService>()
        .BuildServiceProvider();
}


static async Task SyncLdap(IServiceProvider provider)
{
    var context = provider.GetRequiredService<BpmcoreContext>();
    var repo = new LDAPUsersRepository(context);
    var service = provider.GetRequiredService<LdapEmployeeSyncService>(); // ВАЖНО: теперь получаем из DI
    var entries = service.GetLdapEntries();
    var employees = entries.Select(e => new LDAPEmployee(e)).ToList();

    await repo.UpdateByKey(employees);
    Console.WriteLine($"✅ Импортировано: {employees.Count} записей в таблицу ldap_employees");
}
