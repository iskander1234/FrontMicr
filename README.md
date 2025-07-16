Unhandled exception. System.InvalidOperationException: No service for type 'Microsoft.Extensions.Configuration.IConfiguration' has been registered.
   at Microsoft.Extensions.DependencyInjection.ServiceProviderServiceExtensions.GetRequiredService(IServiceProvider provider, Type serviceType)
   at Microsoft.Extensions.DependencyInjection.ServiceProviderServiceExtensions.GetRequiredService[T](IServiceProvider provider)
   at DinDin.Program.SyncLdap(IServiceProvider provider) in C:\BPM\Leshan\1\DinDin\Program.cs:line 127
   at DinDin.Program.StartData(ServiceProvider serviceProvider) in C:\BPM\Leshan\1\DinDin\Program.cs:line 76
   at DinDin.Program.Main() in C:\BPM\Leshan\1\DinDin\Program.cs:line 24
   at DinDin.Program.<Main>()

Process finished with exit code -532,462,766.
