"C:\Program Files\JetBrains\JetBrains Rider 2025.1.2\plugins\dpa\DotFiles\JetBrains.DPA.Runner.exe" --handle=12232 --backend-pid=8868 --etw-collect-flags=67108622 --detach-event-name=dpa.detach.8868.104 --refresh-interval=1 -- C:/BPM/Leshan/1/DinDin/bin/Debug/net8.0/DinDin.exe
warn: Microsoft.EntityFrameworkCore.Model.Validation[10400]
      Sensitive data logging is enabled. Log entries and exception messages may include sensitive application data; this mode should only be enabled during development.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (163ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      CREATE TABLE "public"."DepartmentsTemp9630e61d" AS TABLE "public"."Departments" WITH NO DATA;
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (31ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO "public"."Departments" ("id", "actual", "manager_id", "name", "parent_id") (SELECT "id", "actual", "manager_id", "name", "parent_id" FROM "public"."DepartmentsTemp9630e61d") ON CONFLICT ("id") DO UPDATE SET "actual" = EXCLUDED."actual", "manager_id" = EXCLUDED."manager_id", "name" = EXCLUDED."name", "parent_id" = EXCLUDED."parent_id" RETURNING "public"."Departments"."id", "public"."Departments"."actual", "public"."Departments"."manager_id", "public"."Departments"."name", "public"."Departments"."parent_id"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (34ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      DROP TABLE IF EXISTS "public"."DepartmentsTemp9630e61d"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (11ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      CREATE TABLE "public"."DepartmentsTemp3d83b91e" AS TABLE "public"."Departments" WITH NO DATA;
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (22ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO "public"."Departments" ("id", "actual", "manager_id", "name", "parent_id") (SELECT "id", "actual", "manager_id", "name", "parent_id" FROM "public"."DepartmentsTemp3d83b91e") ON CONFLICT ("id") DO UPDATE SET "actual" = EXCLUDED."actual", "manager_id" = EXCLUDED."manager_id", "name" = EXCLUDED."name", "parent_id" = EXCLUDED."parent_id" RETURNING "public"."Departments"."id", "public"."Departments"."actual", "public"."Departments"."manager_id", "public"."Departments"."name", "public"."Departments"."parent_id"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (2ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      DROP TABLE IF EXISTS "public"."DepartmentsTemp3d83b91e"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (18ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT e.id, e.dep_id, e.dep_name, e.disabled, e.is_filial, e.is_manager, e.local_phone, e.login, e.login_ad, e.mail, e.manager_tab_number, e.mobile_phone, e.name, e.parent_dep_id, e.parent_dep_name, e.position, e.status_code, e.status_description, e.tab_number
      FROM public."Employees" AS e
info: DinDin.Services.LdapEmployeeSyncService[0]
      LDAPConfig: 172.31.0.252:389, BaseDN=dc=enpf,dc=kz
info: DinDin.Services.LdapEmployeeSyncService[0]
      Connecting to LDAP System.DirectoryServices.Protocols.LdapDirectoryIdentifier...
info: DinDin.Services.LdapEmployeeSyncService[0]
      LDAP bind successful
info: DinDin.Services.LdapEmployeeSyncService[0]
      Received page: 1000 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Total LDAP entries retrieved: 1000
info: DinDin.Services.LdapEmployeeSyncService[0]
      Total LDAPEmployee objects created: 1000
?? Retrieved 1000 LDAP records
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (47ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      TRUNCATE TABLE "ldap_employees" RESTART IDENTITY
? Imported 1000 into ldap_employees table
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (4ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT l.id, l.city, l.company, l.department, l.department_details, l.department_id, l.email, l.given_name, l.group_id, l.is_disabled, l.is_manager, l.local_phone, l.location_address, l.login, l.manager, l.member_of, l.member_of_string, l.mobile_number, l.name, l.position_name, l.postal_code, l.title, l.user_status_code
      FROM ldap_employees AS l
      WHERE l.email IS NOT NULL AND l.email <> ''
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (2ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT e.id, e.dep_id, e.dep_name, e.disabled, e.is_filial, e.is_manager, e.local_phone, e.login, e.login_ad, e.mail, e.manager_tab_number, e.mobile_phone, e.name, e.parent_dep_id, e.parent_dep_name, e.position, e.status_code, e.status_description, e.tab_number
      FROM public."Employees" AS e
      WHERE e.mail IS NOT NULL AND e.mail <> ''
?? Updated LoginAd in Employees table

Process finished with exit code 0.

