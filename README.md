"C:\Program Files\JetBrains\JetBrains Rider 2025.1.2\plugins\dpa\DotFiles\JetBrains.DPA.Runner.exe" --handle=9564 --backend-pid=8868 --etw-collect-flags=67108622 --detach-event-name=dpa.detach.8868.51 --refresh-interval=1 -- C:/BPM/Leshan/1/DinDin/bin/Debug/net8.0/DinDin.exe
warn: Microsoft.EntityFrameworkCore.Model.Validation[10400]
      Sensitive data logging is enabled. Log entries and exception messages may include sensitive application data; this mode should only be enabled during development.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (169ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      CREATE TABLE "public"."DepartmentsTemp7aab108f" AS TABLE "public"."Departments" WITH NO DATA;
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (29ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO "public"."Departments" ("id", "actual", "manager_id", "name", "parent_id") (SELECT "id", "actual", "manager_id", "name", "parent_id" FROM "public"."DepartmentsTemp7aab108f") ON CONFLICT ("id") DO UPDATE SET "actual" = EXCLUDED."actual", "manager_id" = EXCLUDED."manager_id", "name" = EXCLUDED."name", "parent_id" = EXCLUDED."parent_id" RETURNING "public"."Departments"."id", "public"."Departments"."actual", "public"."Departments"."manager_id", "public"."Departments"."name", "public"."Departments"."parent_id"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (22ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      DROP TABLE IF EXISTS "public"."DepartmentsTemp7aab108f"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (14ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      CREATE TABLE "public"."DepartmentsTemp3d5a93c1" AS TABLE "public"."Departments" WITH NO DATA;
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (25ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO "public"."Departments" ("id", "actual", "manager_id", "name", "parent_id") (SELECT "id", "actual", "manager_id", "name", "parent_id" FROM "public"."DepartmentsTemp3d5a93c1") ON CONFLICT ("id") DO 
UPDATE SET "actual" = EXCLUDED."actual", "manager_id" = EXCLUDED."manager_id", "name" = EXCLUDED."name", "parent_id" = EXCLUDED."parent_id" RETURNING "public"."Departments"."id", "public"."Departments"."actual", "public"."Departments"."manager_id", "public"."Departments"."name", "public"."Departments"."parent_id"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (2ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      DROP TABLE IF EXISTS "public"."DepartmentsTemp3d5a93c1"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (19ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT e.id, e.dep_id, e.dep_name, e.disabled, e.is_filial, e.is_manager, e.local_phone, e.login, e.mail, e.manager_tab_number, e.mobile_phone, e.name, e.parent_dep_id, e.parent_dep_name, e.position, e.status_code, e.status_description, e.tab_number
      FROM public."Employees" AS e
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
