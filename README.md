"C:\Program Files\JetBrains\JetBrains Rider 2025.1.2\plugins\dpa\DotFiles\JetBrains.DPA.Runner.exe" --handle=3444 --backend-pid=8868 --etw-collect-flags=67108622 --detach-event-name=dpa.detach.8868.90 --refresh-interval=1 -- C:/BPM/Leshan/1/DinDin/bin/Debug/net8.0/DinDin.exe
warn: Microsoft.EntityFrameworkCore.Model.Validation[10400]
      Sensitive data logging is enabled. Log entries and exception messages may include sensitive application data; this mode should only be enabled during development.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (212ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      CREATE TABLE "public"."DepartmentsTemp136bbeb3" AS TABLE "public"."Departments" WITH NO DATA;
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (33ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO "public"."Departments" ("id", "actual", "manager_id", "name", "parent_id") (SELECT "id", "actual", "manager_id", "name", "parent_id" FROM "public"."DepartmentsTemp136bbeb3") ON CONFLICT ("id") DO UPDATE SET "actual" = EXCLUDED."actual", "manager_id" = EXCLUDED."manager_id", "name" = EXCLUDED."name", "parent_id" = EXCLUDED."parent_id" RETURNING "public"."Departments"."id", "public"."Departments"."actual", "public"."Departments"."manager_id", "public"."Departments"."name", "public"."Departments"."parent_id"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (22ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      DROP TABLE IF EXISTS "public"."DepartmentsTemp136bbeb3"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (12ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      CREATE TABLE "public"."DepartmentsTemp736a8352" AS TABLE "public"."Departments" WITH NO DATA;
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (25ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO "public"."Departments" ("id", "actual", "manager_id", "name", "parent_id") (SELECT "id", "actual", "manager_id", "name", "parent_id" FROM "public"."DepartmentsTemp736a835
2") ON CONFLICT ("id") DO UPDATE SET "actual" = EXCLUDED."actual", "manager_id" = EXCLUDED."manager_id", "name" = EXCLUDED."name", "parent_id" = EXCLUDED."parent_id" RETURNING "public"."Departments"."id", "public"."Departments"."actual", "public"."Departments"."manager_id", "public"."Departments"."name", "public"."Departments"."parent_id"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (3ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      DROP TABLE IF EXISTS "public"."DepartmentsTemp736a8352"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (16ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT e.id, e.dep_id, e.dep_name, e.disabled, e.is_filial, e.is_manager, e.local_phone, e.login, e.login_ad, e.mail, e.manager_tab_number, e.mobile_phone, e.name, e.parent_dep_id, e.parent_dep_name, e.position, e.status_code, e.status_description, e.tab_number
      FROM public."Employees" AS e
info: DinDin.Services.LdapEmployeeSyncService[0]
      LDAP Config loaded: Server=172.31.0.252, Port=389, BindDN=gnpfadm@enpf.kz, SearchBase=dc=enpf,dc=kz
info: DinDin.Services.LdapEmployeeSyncService[0]
      Connecting to LDAP server 172.31.0.252:389...
info: DinDin.Services.LdapEmployeeSyncService[0]
      LDAP bind successful
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=0*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment 0: loaded 0 entries, total so far: 0
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=1*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment 1: loaded 3 entries, total so far: 3
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=2*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment 2: loaded 0 entries, total so far: 3
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=3*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment 3: loaded 2 entries, total so far: 5
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=4*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment 4: loaded 0 entries, total so far: 5
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=5*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment 5: loaded 0 entries, total so far: 5
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=6*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment 6: loaded 0 entries, total so far: 5
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=7*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment 7: loaded 0 entries, total so far: 5
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=8*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment 8: loaded 0 entries, total so far: 5
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=9*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment 9: loaded 0 entries, total so far: 5
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=A*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment A: loaded 1000 entries, total so far: 1005
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=B*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment B: loaded 180 entries, total so far: 1185
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=C*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment C: loaded 10 entries, total so far: 1195
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=D*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment D: loaded 239 entries, total so far: 1434
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=E*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment E: loaded 158 entries, total so far: 1592
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=F*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment F: loaded 29 entries, total so far: 1621
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=G*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment G: loaded 341 entries, total so far: 1962
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=H*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment H: loaded 36 entries, total so far: 1998
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=I*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment I: loaded 73 entries, total so far: 2071
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=J*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment J: loaded 11 entries, total so far: 2082
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=K*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment K: loaded 178 entries, total so far: 2260
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=L*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment L: loaded 99 entries, total so far: 2359
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=M*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment M: loaded 324 entries, total so far: 2683
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=N*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment N: loaded 246 entries, total so far: 2929
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=O*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment O: loaded 73 entries, total so far: 3002
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=P*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment P: loaded 22 entries, total so far: 3024
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=Q*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment Q: loaded 3 entries, total so far: 3027
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=R*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment R: loaded 144 entries, total so far: 3171
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=S*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment S: loaded 367 entries, total so far: 3538
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=T*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment T: loaded 109 entries, total so far: 3647
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=U*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment U: loaded 37 entries, total so far: 3684
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=V*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment V: loaded 38 entries, total so far: 3722
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=W*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment W: loaded 0 entries, total so far: 3722
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=X*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment X: loaded 1 entries, total so far: 3723
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=Y*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment Y: loaded 95 entries, total so far: 3818
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=Z*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment Z: loaded 274 entries, total so far: 4092
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=a*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment a: loaded 1000 entries, total so far: 5092
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=b*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment b: loaded 180 entries, total so far: 5272
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=c*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment c: loaded 10 entries, total so far: 5282
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=d*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment d: loaded 239 entries, total so far: 5521
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=e*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment e: loaded 158 entries, total so far: 5679
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=f*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment f: loaded 29 entries, total so far: 5708
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=g*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment g: loaded 341 entries, total so far: 6049
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=h*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment h: loaded 36 entries, total so far: 6085
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=i*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment i: loaded 73 entries, total so far: 6158
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=j*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment j: loaded 11 entries, total so far: 6169
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=k*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment k: loaded 178 entries, total so far: 6347
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=l*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment l: loaded 99 entries, total so far: 6446
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=m*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment m: loaded 324 entries, total so far: 6770
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=n*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment n: loaded 246 entries, total so far: 7016
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=o*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment o: loaded 73 entries, total so far: 7089
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=p*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment p: loaded 22 entries, total so far: 7111
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=q*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment q: loaded 3 entries, total so far: 7114
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=r*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment r: loaded 144 entries, total so far: 7258
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=s*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment s: loaded 367 entries, total so far: 7625
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=t*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment t: loaded 109 entries, total so far: 7734
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=u*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment u: loaded 37 entries, total so far: 7771
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=v*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment v: loaded 38 entries, total so far: 7809
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=w*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment w: loaded 0 entries, total so far: 7809
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=x*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment x: loaded 1 entries, total so far: 7810
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=y*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment y: loaded 95 entries, total so far: 7905
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment filter: (&(&(objectClass=user)(objectCategory=PERSON))(sAMAccountName=z*))
info: DinDin.Services.LdapEmployeeSyncService[0]
      Segment z: loaded 274 entries, total so far: 8179
info: DinDin.Services.LdapEmployeeSyncService[0]
      ?? Removing duplicates by Login... original count: 8179
info: DinDin.Services.LdapEmployeeSyncService[0]
      ? Count after deduplication: 4092
info: DinDin.Services.LdapEmployeeSyncService[0]
      ?? Total unique records retrieved from LDAP: 4092
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (44ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      TRUNCATE TABLE "ldap_employees" RESTART IDENTITY
 Импортировано 4092 записей в таблицу ldap_employees.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (5ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT l.id, l.city, l.company, l.department, l.department_details, l.department_id, l.email, l.given_name, l.group_id, l.is_disabled, l.is_manager, l.local_phone, l.location_address, l.login, l.manager, l.member_of, l.member_of_string, l.mobile_number, l.name, l.position_name, l.postal_code, l.title, l.user_status_code
      FROM ldap_employees AS l
      WHERE l.email IS NOT NULL AND l.email <> ''
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (1ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT e.id, e.dep_id, e.dep_name, e.disabled, e.is_filial, e.is_manager, e.local_phone, e.login, e.login_ad, e.mail, e.manager_tab_number, e.mobile_phone, e.name, e.parent_dep_id, e.parent_dep_name, e.position, e.status_code, e.status_description, e.tab_number
      FROM public."Employees" AS e
      WHERE e.mail IS NOT NULL AND e.mail <> ''
 Обновлены поля LoginAd в таблице employees.

Process finished with exit code 0.

