info: DinDin.Services.LdapEmployeeSyncService[0]
      LDAP Config loaded: Server=172.31.0.252, Port=389, BindDN=gnpfadm@enpf.kz, SearchBase=dc=enpf,dc=kz
info: DinDin.Services.LdapEmployeeSyncService[0]
      Connecting to LDAP server 172.31.0.252:389...
info: DinDin.Services.LdapEmployeeSyncService[0]
      LDAP bind successful
info: DinDin.Services.LdapEmployeeSyncService[0]
      Sending LDAP search request with filter: (&(objectClass=user)(objectCategory=PERSON))
fail: DinDin.Services.LdapEmployeeSyncService[0]
      Unexpected error during LDAP sync
      System.DirectoryServices.Protocols.DirectoryOperationException: An operation error occurred. 000004DC: LdapErr: DSID-0C090A37, comment: In order to perform this operation a successful bind must be completed on the connection., data 0, v4563
         at System.DirectoryServices.Protocols.LdapConnection.ConstructResponseAsync(Int32 messageId, LdapOperation operation, ResultAll resultType, TimeSpan requestTimeOut, Boolean exceptionOnTimeOut, Boolean sync)
         at System.DirectoryServices.Protocols.LdapConnection.SendRequest(DirectoryRequest request, TimeSpan requestTimeout)
         at System.DirectoryServices.Protocols.LdapConnection.SendRequest(DirectoryRequest request)
         at DinDin.Services.LdapEmployeeSyncService.GetLdapEmployees() in C:\BPM\Leshan\1\DinDin\Services\LdapEmployeeSyncService.cs:line 52
Unhandled exception. System.DirectoryServices.Protocols.DirectoryOperationException: An operation error occurred. 000004DC: LdapErr: DSID-0C090A37, comment: In order to perform this operation a successful bind must be completed on the connection., data 0, v4563
   at System.DirectoryServices.Protocols.LdapConnection.ConstructResponseAsync(Int32 messageId, LdapOperation operation, ResultAll resultType, TimeSpan requestTimeOut, Boolean exceptionOnTimeOut, Boolean sync)     
   at System.DirectoryServices.Protocols.LdapConnection.SendRequest(DirectoryRequest request, TimeSpan requestTimeout)
   at System.DirectoryServices.Protocols.LdapConnection.SendRequest(DirectoryRequest request)
   at DinDin.Services.LdapEmployeeSyncService.GetLdapEmployees() in C:\BPM\Leshan\1\DinDin\Services\LdapEmployeeSyncService.cs:line 52
   at DinDin.Program.SyncLdap(IServiceProvider provider) in C:\BPM\Leshan\1\DinDin\Program.cs:line 139
   at DinDin.Program.StartData(ServiceProvider serviceProvider) in C:\BPM\Leshan\1\DinDin\Program.cs:line 77
   at DinDin.Program.Main() in C:\BPM\Leshan\1\DinDin\Program.cs:line 25
   at DinDin.Program.<Main>()

Process finished with exit code -532,462,766.
