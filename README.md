Unhandled exception. System.DirectoryServices.Protocols.LdapException: The supplied credential is invalid.
   at System.DirectoryServices.Protocols.LdapConnection.BindHelper(NetworkCredential newCredential, Boolean needSetCredential)
   at System.DirectoryServices.Protocols.LdapConnection.Bind(NetworkCredential newCredential)
   at DinDin.Services.LdapEmployeeSyncService.GetLdapEmployees() in C:\BPM\Leshan\1\DinDin\Services\LdapEmployeeSyncService.cs:line 21
   at DinDin.Program.SyncLdap(IServiceProvider provider) in C:\BPM\Leshan\1\DinDin\Program.cs:line 129
   at DinDin.Program.StartData(ServiceProvider serviceProvider) in C:\BPM\Leshan\1\DinDin\Program.cs:line 76
   at DinDin.Program.Main() in C:\BPM\Leshan\1\DinDin\Program.cs:line 24
   at DinDin.Program.<Main>()
