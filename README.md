 return new ServiceCollection()
        .AddSingleton<IConfiguration>(configuration)
        .AddDbContext<BpmcoreContext>(options => options.UseNpgsql(connectionString))
        .AddSingleton<OneCService>()
        .AddSingleton<DepartmentRepository>()
        .AddSingleton<EmployeeRepository>()
        .AddSingleton<LdapEmployeeSyncService>()
        .BuildServiceProvider();
