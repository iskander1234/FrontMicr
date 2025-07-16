private static async Task SyncLdap(IServiceProvider provider)
{
    var context = provider.GetRequiredService<BpmcoreContext>();
    var ldapService = provider.GetRequiredService<LdapEmployeeSyncService>();
    var ldapUsersRepo = provider.GetRequiredService<LDAPUsersRepository>();

    var ldapEmployees = ldapService.GetLdapEmployees();

    await ldapUsersRepo.UpdateByKey(ldapEmployees);

    Console.WriteLine($"✅ Импортировано {ldapEmployees.Count} записей в таблицу ldap_employees.");
}


private static ServiceProvider ConfigureServices(IConfiguration configuration)
{
    var connectionString = configuration.GetConnectionString("BpmCore") ?? throw new Exception("ConnectionString BpmCore was not found");

    return new ServiceCollection()
        .AddDbContext<BpmcoreContext>(options => options.UseNpgsql(connectionString))
        .AddSingleton(configuration) // добавь IConfiguration в DI
        .AddSingleton<OneCService>()
        .AddSingleton<DepartmentRepository>()
        .AddSingleton<EmployeeRepository>()
        .AddSingleton<LdapEmployeeSyncService>()   // 👈 Добавь эту строку
        .AddSingleton<LDAPUsersRepository>()       // 👈 И эту
        .BuildServiceProvider();
}
