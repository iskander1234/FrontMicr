"C:\Program Files\JetBrains\JetBrains Rider 2025.1.2\plugins\dpa\DotFiles\JetBrains.DPA.Runner.exe" --handle=10432 --backend-pid=8868 --etw-collect-flags=67108622 --detach-event-name=dpa.detach.8868.91 --refresh-interval=1 -- C:/BPM/Leshan/1/DinDin/bin/Debug/net8.0/DinDin.exe
warn: Microsoft.EntityFrameworkCore.Model.Validation[10400]
      Sensitive data logging is enabled. Log entries and exception messages may include sensitive application data; this mode should only be enabled during development.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (196ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      CREATE TABLE "public"."DepartmentsTempf14a377e" AS TABLE "public"."Departments" WITH NO DATA;
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (33ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO "public"."Departments" ("id", "actual", "manager_id", "name", "parent_id") (SELECT "id", "actual", "manager_id", "name", "parent_id" FROM "public"."DepartmentsTempf14a377e") ON CONFLICT ("id") DO UPDATE SET "actual" = EXCLUDED."actual", "manager_id" = EXCLUDED."manager_id", "name" = EXCLUDED."name", "parent_id" = EXCLUDED."parent_id" RETURNING "public"."Departments"."id", "public"."Departments"."actual", "public"."Departments"."manager_id", "public"."Departments"."name", "public"."Departments"."parent_id"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (22ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      DROP TABLE IF EXISTS "public"."DepartmentsTempf14a377e"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (13ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      CREATE TABLE "public"."DepartmentsTempbbfe46df" AS TABLE "public"."Departments" WITH NO DATA;
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (23ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO "public"."Departments" ("id", "actual", "manager_id", "name", "parent_id") (SELECT "id", "actual", "manager_id", "name", "parent_id" FROM "public"."DepartmentsTempbbfe46d
f") ON CONFLICT ("id") DO UPDATE SET "actual" = EXCLUDED."actual", "manager_id" = EXCLUDED."manager_id", "name" = EXCLUDED."name", "parent_id" = EXCLUDED."parent_id" RETURNING "public"."Departments"."id", "public"."Departments"."actual", "public"."Departments"."manager_id", "public"."Departments"."name", "public"."Departments"."parent_id"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (2ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      DROP TABLE IF EXISTS "public"."DepartmentsTempbbfe46df"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (16ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT e.id, e.dep_id, e.dep_name, e.disabled, e.is_filial, e.is_manager, e.local_phone, e.login, e.login_ad, e.mail, e.manager_tab_number, e.mobile_phone, e.name, e.parent_dep_id, e.parent_dep_name, e.position, e.status_code, e.status_description, e.tab_number
      FROM public."Employees" AS e
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (16ms) [Parameters=[@p2='2793', @p0='1' (Nullable = true), @p1='Болезнь'], CommandType='Text', CommandTimeout='30']
      UPDATE public."Employees" SET status_code = @p0, status_description = @p1
      WHERE id = @p2;
info: DinDin.Services.LdapEmployeeSyncService[0]
      LDAP Config loaded: Server=172.31.0.252, Port=389, BindDN=gnpfadm@enpf.kz, SearchBase=dc=enpf,dc=kz
info: DinDin.Services.LdapEmployeeSyncService[0]
      Connecting to LDAP via DirectorySearcher: 172.31.0.252:389...
info: DinDin.Services.LdapEmployeeSyncService[0]
      Starting DirectorySearcher.FindAll()...
info: DinDin.Services.LdapEmployeeSyncService[0]
      DirectorySearcher returned 4233 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      ?? Total records retrieved from LDAP: 0
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (48ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      TRUNCATE TABLE "ldap_employees" RESTART IDENTITY
 Импортировано 0 записей в таблицу ldap_employees.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (4ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT l.id, l.city, l.company, l.department, l.department_details, l.department_id, l.email, l.given_name, l.group_id, l.is_disabled, l.is_manager, l.local_phone, l.location_address, l.login, l.manager, l.member_of, l.member_of_string, l.mobile_number, l.name, l.position_name, l.postal_code, l.title, l.user_status_code
      FROM ldap_employees AS l
      WHERE l.email IS NOT NULL AND l.email <> ''
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (12ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT e.id, e.dep_id, e.dep_name, e.disabled, e.is_filial, e.is_manager, e.local_phone, e.login, e.login_ad, e.mail, e.manager_tab_number, e.mobile_phone, e.name, e.parent_dep_id, e.parent_dep_name, e.position, e.status_code, e.status_description, e.tab_number
      FROM public."Employees" AS e
      WHERE e.mail IS NOT NULL AND e.mail <> ''
 Обновлены поля LoginAd в таблице employees.

Process finished with exit code 0.

