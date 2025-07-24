"C:\Program Files\JetBrains\JetBrains Rider 2025.1.2\plugins\dpa\DotFiles\JetBrains.DPA.Runner.exe" --handle=4352 --backend-pid=8868 --etw-collect-flags=67108622 --detach-event-name=dpa.detach.8868.84 --refresh-interval=1 -- C:/BPM/Leshan/1/DinDin/bin/Debug/net8.0/DinDin.exe
warn: Microsoft.EntityFrameworkCore.Model.Validation[10400]
      Sensitive data logging is enabled. Log entries and exception messages may include sensitive application data; this mode should only be enabled during development.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (155ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      CREATE TABLE "public"."DepartmentsTempa2a5309c" AS TABLE "public"."Departments" WITH NO DATA;
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (27ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO "public"."Departments" ("id", "actual", "manager_id", "name", "parent_id") (SELECT "id", "actual", "manager_id", "name", "parent_id" FROM "public"."DepartmentsTempa2a5309c") ON CONFLICT ("id") DO UPDATE SET "actual" = EXCLUDED."actual", "manager_id" = EXCLUDED."manager_id", "name" = EXCLUDED."name", "parent_id" = EXCLUDED."parent_id" RETURNING "public"."Departments"."id", "public"."Departments"."actual", "public"."Departments"."manager_id", "public"."Departments"."name", "public"."Departments"."parent_id"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (20ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      DROP TABLE IF EXISTS "public"."DepartmentsTempa2a5309c"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (5ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      CREATE TABLE "public"."DepartmentsTemp2b88c29b" AS TABLE "public"."Departments" WITH NO DATA;
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (7ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO "public"."Departments" ("id", "actual", "manager_id", "name", "parent_id") (SELECT "id", "actual", "manager_id", "name", "parent_id" FROM "public"."DepartmentsTemp2b88c29
b") ON CONFLICT ("id") DO UPDATE SET "actual" = EXCLUDED."actual", "manager_id" = EXCLUDED."manager_id", "name" = EXCLUDED."name", "parent_id" = EXCLUDED."parent_id" RETURNING "public"."Departments"."id", "public"."Departments"."actual", "public"."Departments"."manager_id", "public"."Departments"."name", "public"."Departments"."parent_id"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (1ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      DROP TABLE IF EXISTS "public"."DepartmentsTemp2b88c29b"
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (16ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT e.id, e.dep_id, e.dep_name, e.disabled, e.is_filial, e.is_manager, e.local_phone, e.login, e.login_ad, e.mail, e.manager_tab_number, e.mobile_phone, e.name, e.parent_dep_id, e.parent_dep_name, e.position, e.status_code, e.status_description, e.tab_number
      FROM public."Employees" AS e
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (21ms) [Parameters=[@p2='31', @p0='6' (Nullable = true), @p1='Работа', @p5='66', @p3='5' (Nullable = true), @p4='Отпуск основной', @p8='114', @p6='1' (Nullable = t
rue), @p7='Болезнь', @p11='136', @p9='6' (Nullable = true), @p10='Работа', @p14='187', @p12='1' (Nullable = true), @p13='Болезнь', @p17='237', @p15='6' (Nullable = true), @p16='Работа', @p
20='253', @p18='6' (Nullable = true), @p19='Работа', @p23='275', @p21='3' (Nullable = true), @p22='Командировка', @p26='356', @p24='1' (Nullable = true), @p25='Болезнь', @p29='358', @p27='
6' (Nullable = true), @p28='Работа', @p32='396', @p30='3' (Nullable = true), @p31='Командировка', @p35='472', @p33='6' (Nullable = true), @p34='Работа', @p38='533', @p36='6' (Nullable = tr
ue), @p37='Работа', @p41='544', @p39='8' (Nullable = true), @p40='Отпуск неоплачиваемый по законодательству', @p44='577', @p42='3' (Nullable = true), @p43='Командировка', @p47='619', @p45=
'6' (Nullable = true), @p46='Работа', @p50='647', @p48='5' (Nullable = true), @p49='Отпуск основной', @p53='660', @p51='1' (Nullable = true), @p52='Болезнь', @p56='702', @p54='6' (Nullable
 = true), @p55='Работа', @p59='733', @p57='1' (Nullable = true), @p58='Болезнь', @p62='768', @p60='6' (Nullable = true), @p61='Работа', @p65='1240', @p63='6' (Nullable = true), @p64='Работ
а', @p68='1297', @p66='1' (Nullable = true), @p67='Болезнь', @p71='1381', @p69='1' (Nullable = true), @p70='Болезнь', @p74='1422', @p72='1' (Nullable = true), @p73='Болезнь', @p77='1423', 
@p75='11' (Nullable = true), @p76='Отпуск по беременности и родам', @p80='1431', @p78='1' (Nullable = true), @p79='Болезнь', @p83='1476', @p81='1' (Nullable = true), @p82='Болезнь', @p86='
1495', @p84='8' (Nullable = true), @p85='Отпуск неоплачиваемый по законодательству', @p93='1506', @p87='18.018150', @p88='Отделение, ул.Байсеитова, 29', @p89='511', @p90='18.018001', @p91=
'Областной филиал Абай', @p92='Ведущий специалист', @p96='1520', @p94='6' (Nullable = true), @p95='Работа', @p99='1558', @p97='z.kadyr', @p98='z.kadyr@enpf.kz', @p102='1563', @p100='z.suiimbek', @p101='z.suiimbek@enpf.kz'], CommandType='Text', CommandTimeout='30']
      UPDATE public."Employees" SET status_code = @p0, status_description = @p1
      WHERE id = @p2;
      UPDATE public."Employees" SET status_code = @p3, status_description = @p4
      WHERE id = @p5;
      UPDATE public."Employees" SET status_code = @p6, status_description = @p7
      WHERE id = @p8;
      UPDATE public."Employees" SET status_code = @p9, status_description = @p10
      WHERE id = @p11;
      UPDATE public."Employees" SET status_code = @p12, status_description = @p13
      WHERE id = @p14;
      UPDATE public."Employees" SET status_code = @p15, status_description = @p16
      WHERE id = @p17;
      UPDATE public."Employees" SET status_code = @p18, status_description = @p19
      WHERE id = @p20;
      UPDATE public."Employees" SET status_code = @p21, status_description = @p22
      WHERE id = @p23;
      UPDATE public."Employees" SET status_code = @p24, status_description = @p25
      WHERE id = @p26;
      UPDATE public."Employees" SET status_code = @p27, status_description = @p28
      WHERE id = @p29;
      UPDATE public."Employees" SET status_code = @p30, status_description = @p31
      WHERE id = @p32;
      UPDATE public."Employees" SET status_code = @p33, status_description = @p34
      WHERE id = @p35;
      UPDATE public."Employees" SET status_code = @p36, status_description = @p37
      WHERE id = @p38;
      UPDATE public."Employees" SET status_code = @p39, status_description = @p40
      WHERE id = @p41;
      UPDATE public."Employees" SET status_code = @p42, status_description = @p43
      WHERE id = @p44;
      UPDATE public."Employees" SET status_code = @p45, status_description = @p46
      WHERE id = @p47;
      UPDATE public."Employees" SET status_code = @p48, status_description = @p49
      WHERE id = @p50;
      UPDATE public."Employees" SET status_code = @p51, status_description = @p52
      WHERE id = @p53;
      UPDATE public."Employees" SET status_code = @p54, status_description = @p55
      WHERE id = @p56;
      UPDATE public."Employees" SET status_code = @p57, status_description = @p58
      WHERE id = @p59;
      UPDATE public."Employees" SET status_code = @p60, status_description = @p61
      WHERE id = @p62;
      UPDATE public."Employees" SET status_code = @p63, status_description = @p64
      WHERE id = @p65;
      UPDATE public."Employees" SET status_code = @p66, status_description = @p67
      WHERE id = @p68;
      UPDATE public."Employees" SET status_code = @p69, status_description = @p70
      WHERE id = @p71;
      UPDATE public."Employees" SET status_code = @p72, status_description = @p73
      WHERE id = @p74;
      UPDATE public."Employees" SET status_code = @p75, status_description = @p76
      WHERE id = @p77;
      UPDATE public."Employees" SET status_code = @p78, status_description = @p79
      WHERE id = @p80;
      UPDATE public."Employees" SET status_code = @p81, status_description = @p82
      WHERE id = @p83;
      UPDATE public."Employees" SET status_code = @p84, status_description = @p85
      WHERE id = @p86;
      UPDATE public."Employees" SET dep_id = @p87, dep_name = @p88, manager_tab_number = @p89, parent_dep_id = @p90, parent_dep_name = @p91, position = @p92
      WHERE id = @p93;
      UPDATE public."Employees" SET status_code = @p94, status_description = @p95
      WHERE id = @p96;
      UPDATE public."Employees" SET login = @p97, mail = @p98
      WHERE id = @p99;
      UPDATE public."Employees" SET login = @p100, mail = @p101
      WHERE id = @p102;
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
      ?? Total records retrieved from LDAP: 4092
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (12ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      TRUNCATE TABLE "ldap_employees" RESTART IDENTITY
 Импортировано 4092 записей в таблицу ldap_employees.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (1ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT l.id, l.city, l.company, l.department, l.department_details, l.department_id, l.email, l.given_name, l.group_id, l.is_disabled, l.is_manager, l.local_phone, l.location_address, l.login, l.manager, l.member_of, l.member_of_string, l.mobile_number, l.name, l.position_name, l.postal_code, l.title, l.user_status_code
      FROM ldap_employees AS l
      WHERE l.email IS NOT NULL AND l.email <> ''
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (2ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT e.id, e.dep_id, e.dep_name, e.disabled, e.is_filial, e.is_manager, e.local_phone, e.login, e.login_ad, e.mail, e.manager_tab_number, e.mobile_phone, e.name, e.parent_dep_id, e.parent_dep_name, e.position, e.status_code, e.status_description, e.tab_number
      FROM public."Employees" AS e
      WHERE e.mail IS NOT NULL AND e.mail <> ''
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (12ms) [Parameters=[@p1='1', @p0='a.abayeva', @p3='66', @p2='u.azhigulova', @p5='112', @p4='d.alibekov', @p7='222', @p6='A.baibulat', @p9='226', @p8='D.Baidildaeva
', @p11='281', @p10='L.Begmuratova', @p13='287', @p12='A.Beisenbay', @p15='305', @p14='n.belova', @p17='312', @p16='y.berikuly', @p19='354', @p18='M.Bychkov', @p21='365', @p20='g.gafarova'
, @p23='377', @p22='S.Gryshaeva', @p25='399', @p24='Zh.jarimbetova', @p27='506', @p26='L.Zhakyp', @p29='536', @p28='e.zheldybayev', @p31='544', @p30='d.zhilkaidarova', @p33='572', @p32='za
.zhunusova', @p35='627', @p34='d.issakhanova', @p37='828', @p36='s.kuznetsova', @p39='836', @p38='A.Kulumbetova', @p41='868', @p40='L.Kuttykozhanova', @p43='871', @p42='A.Kurmangali', @p45
='876', @p44='i.ligay', @p47='881', @p46='A.Alimhan', @p49='893', @p48='s.makasheva', @p51='898', @p50='M.Makulbekov', @p53='945', @p52='E.Moskvina', @p55='966', @p54='m.murzakhalim', @p57
='979', @p56='di.ramazanova', @p59='1021', @p58='L.Naderkulova', @p61='1077', @p60='E.Nurlanov', @p63='1112', @p62='A.Ospangaliyeva', @p65='1154', @p64='Zh.Rashydova', @p67='1344', @p66='G
.Telgaziyeva', @p69='1360', @p68='K.Tkishev', @p71='1382', @p70='z.toleubekova', @p73='1393', @p72='z.tulegenova', @p75='1525', @p74='A.Shashkenova', @p77='1538', @p76='S.Shortanbayeva', @
p79='1540', @p78='I.Shunkova', @p81='1552', @p80='b.altynbekov', @p83='1553', @p82='n.akhmettayev', @p85='1554', @p84='s.bakirova', @p87='1556', @p86='d.yelekeshev', @p89='1557', @p88='a.z
hundubayeva', @p91='1558', @p90='z.kadyr', @p93='1561', @p92='s.marat', @p95='1563', @p94='z.suiimbek', @p97='1564', @p96='s.torekhanuly', @p99='1565', @p98='b.turdalina'], CommandType='Text', CommandTimeout='30']
      UPDATE public."Employees" SET login_ad = @p0
      WHERE id = @p1;
      UPDATE public."Employees" SET login_ad = @p2
      WHERE id = @p3;
      UPDATE public."Employees" SET login_ad = @p4
      WHERE id = @p5;
      UPDATE public."Employees" SET login_ad = @p6
      WHERE id = @p7;
      UPDATE public."Employees" SET login_ad = @p8
      WHERE id = @p9;
      UPDATE public."Employees" SET login_ad = @p10
      WHERE id = @p11;
      UPDATE public."Employees" SET login_ad = @p12
      WHERE id = @p13;
      UPDATE public."Employees" SET login_ad = @p14
      WHERE id = @p15;
      UPDATE public."Employees" SET login_ad = @p16
      WHERE id = @p17;
      UPDATE public."Employees" SET login_ad = @p18
      WHERE id = @p19;
      UPDATE public."Employees" SET login_ad = @p20
      WHERE id = @p21;
      UPDATE public."Employees" SET login_ad = @p22
      WHERE id = @p23;
      UPDATE public."Employees" SET login_ad = @p24
      WHERE id = @p25;
      UPDATE public."Employees" SET login_ad = @p26
      WHERE id = @p27;
      UPDATE public."Employees" SET login_ad = @p28
      WHERE id = @p29;
      UPDATE public."Employees" SET login_ad = @p30
      WHERE id = @p31;
      UPDATE public."Employees" SET login_ad = @p32
      WHERE id = @p33;
      UPDATE public."Employees" SET login_ad = @p34
      WHERE id = @p35;
      UPDATE public."Employees" SET login_ad = @p36
      WHERE id = @p37;
      UPDATE public."Employees" SET login_ad = @p38
      WHERE id = @p39;
      UPDATE public."Employees" SET login_ad = @p40
      WHERE id = @p41;
      UPDATE public."Employees" SET login_ad = @p42
      WHERE id = @p43;
      UPDATE public."Employees" SET login_ad = @p44
      WHERE id = @p45;
      UPDATE public."Employees" SET login_ad = @p46
      WHERE id = @p47;
      UPDATE public."Employees" SET login_ad = @p48
      WHERE id = @p49;
      UPDATE public."Employees" SET login_ad = @p50
      WHERE id = @p51;
      UPDATE public."Employees" SET login_ad = @p52
      WHERE id = @p53;
      UPDATE public."Employees" SET login_ad = @p54
      WHERE id = @p55;
      UPDATE public."Employees" SET login_ad = @p56
      WHERE id = @p57;
      UPDATE public."Employees" SET login_ad = @p58
      WHERE id = @p59;
      UPDATE public."Employees" SET login_ad = @p60
      WHERE id = @p61;
      UPDATE public."Employees" SET login_ad = @p62
      WHERE id = @p63;
      UPDATE public."Employees" SET login_ad = @p64
      WHERE id = @p65;
      UPDATE public."Employees" SET login_ad = @p66
      WHERE id = @p67;
      UPDATE public."Employees" SET login_ad = @p68
      WHERE id = @p69;
      UPDATE public."Employees" SET login_ad = @p70
      WHERE id = @p71;
      UPDATE public."Employees" SET login_ad = @p72
      WHERE id = @p73;
      UPDATE public."Employees" SET login_ad = @p74
      WHERE id = @p75;
      UPDATE public."Employees" SET login_ad = @p76
      WHERE id = @p77;
      UPDATE public."Employees" SET login_ad = @p78
      WHERE id = @p79;
      UPDATE public."Employees" SET login_ad = @p80
      WHERE id = @p81;
      UPDATE public."Employees" SET login_ad = @p82
      WHERE id = @p83;
      UPDATE public."Employees" SET login_ad = @p84
      WHERE id = @p85;
      UPDATE public."Employees" SET login_ad = @p86
      WHERE id = @p87;
      UPDATE public."Employees" SET login_ad = @p88
      WHERE id = @p89;
      UPDATE public."Employees" SET login_ad = @p90
      WHERE id = @p91;
      UPDATE public."Employees" SET login_ad = @p92
      WHERE id = @p93;
      UPDATE public."Employees" SET login_ad = @p94
      WHERE id = @p95;
      UPDATE public."Employees" SET login_ad = @p96
      WHERE id = @p97;
      UPDATE public."Employees" SET login_ad = @p98
      WHERE id = @p99;
 Обновлены поля LoginAd в таблице employees.

Process finished with exit code 0.

