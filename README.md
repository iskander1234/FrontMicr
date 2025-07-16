public static void Main(string[] args)
    {
        var host = Host.CreateDefaultBuilder(args)
            .ConfigureAppConfiguration((context, config) =>
            {
                config.AddJsonFile("appsettings.json", optional: false, reloadOnChange: true);
            })
            .ConfigureServices((context, services) =>
            {
                IConfiguration configuration = context.Configuration;
                services.AddSingleton(configuration);

                services.AddLogging(logging =>
                {
                    logging.AddConsole();
                    logging.SetMinimumLevel(LogLevel.Information);
                });

                // Регистрируем твой сервис
                services.AddTransient<LdapEmployeeSyncService>();
            })
            .Build();

        var service = host.Services.GetRequiredService<LdapEmployeeSyncService>();
        service.GetLdapEmployees(); // или вызов нужного метода
    }
