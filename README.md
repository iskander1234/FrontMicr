"C:\Program Files\JetBrains\JetBrains Rider 2025.1.2\plugins\dpa\DotFiles\JetBrains.DPA.Runner.exe" --handle=10432 --backend-pid=8868 --etw-collect-flags=67108622 --detach-event-name=dpa.detach.8868.79 --refresh-interval=1 -- C:/BPM/Leshan/1/DinDin/bin/Debug/net8.0/DinDin.exe
warn: Microsoft.EntityFrameworkCore.Model.Validation[10400]
      Sensitive data logging is enabled. Log entries and exception messages may include sensitive application data; this mode should only be enabled during development.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (188ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      CREATE TABLE "public"."DepartmentsTemp9de599a7" AS TABLE "public"."Departments" WITH NO DATA;
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (38ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO "public"."Departments" ("id", "actual", "manager_id", "name", "parent_id") (SELECT "id", "actual", "manager_id", "name", "parent_id" FROM "public"."DepartmentsTemp9de599a7") ON CONFLICT ("id") DO UPDATE SET "actual" = EXCLUDED."actual", "manager_id" = EXCLUDED."manager_id", "name" = EXCLUDED."name", "parent_id" = EXCLUDED."parent_id" RETURNING "public"."Departments"."id", "public"."Departments"."actual", "public"."Departments"."manager_id", "public"."Departments"."name", "public"."Departments"."parent_id"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (44ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      DROP TABLE IF EXISTS "public"."DepartmentsTemp9de599a7"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (13ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      CREATE TABLE "public"."DepartmentsTemp4b560a43" AS TABLE "public"."Departments" WITH NO DATA;
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (18ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO "public"."Departments" ("id", "actual", "manager_id", "name", "parent_id") (SELECT "id", "actual", "manager_id", "n
ame", "parent_id" FROM "public"."DepartmentsTemp4b560a43") ON CONFLICT ("id") DO UPDATE SET "actual" = EXCLUDED."actual", "manager_id
" = EXCLUDED."manager_id", "name" = EXCLUDED."name", "parent_id" = EXCLUDED."parent_id" RETURNING "public"."Departments"."id", "public"."Departments"."actual", "public"."Departments"."manager_id", "public"."Departments"."name", "public"."Departments"."parent_id"    
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (5ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      DROP TABLE IF EXISTS "public"."DepartmentsTemp4b560a43"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (18ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT e.id, e.dep_id, e.dep_name, e.disabled, e.is_filial, e.is_manager, e.local_phone, e.login, e.login_ad, e.mail, e.manager
_tab_number, e.mobile_phone, e.name, e.parent_dep_id, e.parent_dep_name, e.position, e.status_code, e.status_description, e.tab_number
      FROM public."Employees" AS e
info: DinDin.Services.LdapEmployeeSyncService[0]
      LDAP Config loaded: Server=172.31.0.252, Port=389, BindDN=gnpfadm@enpf.kz, SearchBase=dc=enpf,dc=kz
info: DinDin.Services.LdapEmployeeSyncService[0]
      Connecting to LDAP server 172.31.0.252:389...
info: DinDin.Services.LdapEmployeeSyncService[0]
      LDAP bind successful
info: DinDin.Services.LdapEmployeeSyncService[0]
      Cookie length: 0
info: DinDin.Services.LdapEmployeeSyncService[0]
      No more pages (cookie length is zero). Ending loop.
info: DinDin.Services.LdapEmployeeSyncService[0]
      ?? Total records received from LDAP: 1000
info: DinDin.Services.LdapEmployeeSyncService[0]
      ? Total imported from LDAP: 1000 records
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (40ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      TRUNCATE TABLE "ldap_employees" RESTART IDENTITY
 Импортировано 1000 записей в таблицу ldap_employees.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (4ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT l.id, l.city, l.company, l.department, l.department_details, l.department_id, l.email, l.given_name, l.group_id, l.is_di
sabled, l.is_manager, l.local_phone, l.location_address, l.login, l.manager, l.member_of, l.member_of_string, l.mobile_number, l.name, l.position_name, l.postal_code, l.title, l.user_status_code
      FROM ldap_employees AS l
      WHERE l.email IS NOT NULL AND l.email <> ''
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (1ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT e.id, e.dep_id, e.dep_name, e.disabled, e.is_filial, e.is_manager, e.local_phone, e.login, e.login_ad, e.mail, e.manager
_tab_number, e.mobile_phone, e.name, e.parent_dep_id, e.parent_dep_name, e.position, e.status_code, e.status_description, e.tab_number
      FROM public."Employees" AS e
      WHERE e.mail IS NOT NULL AND e.mail <> ''
 Обновлены поля LoginAd в таблице employees.

Process finished with exit code 0.

