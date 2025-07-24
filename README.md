C:\Program Files\JetBrains\JetBrains Rider 2025.1.2\plugins\dpa\DotFiles\JetBrains.DPA.Runner.exe" --handle=8640 --backend-pid=8868 --etw-collect-flags=67108622 --detach-event-name=dpa.detach.8868.99 --refresh-interval=1 -- C:/BPM/Leshan/1/DinDin/bin/Debug/net8.0/DinDin.exe
warn: Microsoft.EntityFrameworkCore.Model.Validation[10400]
      Sensitive data logging is enabled. Log entries and exception messages may include sensitive application data; this mode should only be enabled during development.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (152ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      CREATE TABLE "public"."DepartmentsTemp49b6753c" AS TABLE "public"."Departments" WITH NO DATA;
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (27ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO "public"."Departments" ("id", "actual", "manager_id", "name", "parent_id") (SELECT "id", "actual", "manager_id", "name", "parent_id" FROM "public"."DepartmentsTemp49b6753c") ON CONFLICT ("id") DO UPDATE SET "actual" = EXCLUDED."actual", "manager_id" = EXCLUDED."manager_id", "name" = EXCLUDED."name", "parent_id" = EXCLUDED."parent_id" RETURNING "public"."Departments"."id", "public"."Departments"."actual", "public"."Departments"."manager_id", "public"."Departments"."name", "public"."Departments"."parent_id"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (19ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      DROP TABLE IF EXISTS "public"."DepartmentsTemp49b6753c"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (13ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      CREATE TABLE "public"."DepartmentsTempff880718" AS TABLE "public"."Departments" WITH NO DATA;
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (22ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO "public"."Departments" ("id", "actual", "manager_id", "name", "parent_id") (SELECT "id", "actual", "manager_id", "name", "parent_id" FROM "public"."DepartmentsTempff880718") ON CONFLICT ("id") DO UPDATE SET "actual" = EXCLUDED."actual", "manager_id" = EXCLUDED."manager_id", "name" = EXCLUDED."name", "parent_id" = EXCLUDED."parent_id" RETURNING "public"."Departments"."id", "public"."Departments"."actual", "public"."Departments"."manager_id", "public"."Departments"."name", "public"."Departments"."parent_id"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (2ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      DROP TABLE IF EXISTS "public"."DepartmentsTempff880718"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (17ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT e.id, e.dep_id, e.dep_name, e.disabled, e.is_filial, e.is_manager, e.local_phone, e.login, e.login_ad, e.mail, e.manager_tab_number, e.mobile_phone, e.name, e.parent_dep_id, e.parent_dep_name, e.position, e.status_code, e.status_description, e.tab_number
      FROM public."Employees" AS e
info: DinDin.Services.LdapEmployeeSyncService[0]
      LDAP service configured for 172.31.0.252:389, BaseDN=dc=enpf,dc=kz
info: DinDin.Services.LdapEmployeeSyncService[0]
      LDAP bind successful
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=0*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment 0: loaded 0 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=1*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment 1: loaded 3 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=2*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment 2: loaded 0 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=3*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment 3: loaded 2 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=4*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment 4: loaded 0 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=5*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment 5: loaded 0 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=6*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment 6: loaded 0 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=7*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment 7: loaded 0 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=8*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment 8: loaded 0 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=9*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment 9: loaded 0 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=A*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment A: loaded 1000 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=B*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment B: loaded 180 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=C*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment C: loaded 10 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=D*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment D: loaded 239 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=E*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment E: loaded 158 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=F*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment F: loaded 29 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=G*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment G: loaded 341 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=H*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment H: loaded 36 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=I*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment I: loaded 73 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=J*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment J: loaded 11 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=K*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment K: loaded 178 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=L*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment L: loaded 99 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=M*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment M: loaded 324 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=N*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment N: loaded 246 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=O*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment O: loaded 73 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=P*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment P: loaded 22 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=Q*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment Q: loaded 3 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=R*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment R: loaded 144 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=S*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment S: loaded 367 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=T*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment T: loaded 109 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=U*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment U: loaded 37 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=V*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment V: loaded 38 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=W*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment W: loaded 0 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=X*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment X: loaded 1 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=Y*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment Y: loaded 95 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=Z*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment Z: loaded 274 entries
info: DinDin.Services.LdapEmployeeSyncService[0]
      ?? Total unique records retrieved: 4092
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (39ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      TRUNCATE TABLE "ldap_employees" RESTART IDENTITY
 Импортировано 4092 записей в таблицу ldap_employees.
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
 Обновлены поля LoginAd в таблице employees.
