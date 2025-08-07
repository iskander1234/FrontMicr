"C:\Program Files\JetBrains\JetBrains Rider 2025.1.2\plugins\dpa\DotFiles\JetBrains.DPA.Runner.exe" --handle=2180 --backend-pid=12376 --etw-collect-flags=67108622 --detach-event-name=dpa.detach.12376.65 --refresh-interval=1 -- C:/BPM/bpm/bpmbaseapi/BpmBaseApi/bin/Debug/net8.0/BpmBaseApi.exe
warn: Microsoft.EntityFrameworkCore.Model.Validation[10400]
      Sensitive data logging is enabled. Log entries and exception messages may include sensitive application data; this mode should only be enabled during development.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (84ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT "MigrationId", "ProductVersion"
      FROM "__EFMigrationsHistory"
      ORDER BY "MigrationId";
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://localhost:5143
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Development
info: Microsoft.Hosting.Lifetime[0]
      Content root path: C:\BPM\bpm\bpmbaseapi\BpmBaseApi
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (57ms) [Parameters=[@__command_ProcessCode_0='Memo'], CommandType='Text', CommandTimeout='30']
      SELECT p."Id", p."CreateUserId", p."Created", p."LastUserId", p."ProcessCode", p."ProcessName", p."StartBlockId", p."Updated"
      FROM "Process" AS p
      WHERE p."ProcessCode" = @__command_ProcessCode_0
      LIMIT 1
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (8ms) [Parameters=[@__processCode_0='Memo', @__year_1='2025'], CommandType='Text', CommandTimeout='30']
      SELECT r."Id", r."CreateUserId", r."Created", r."LastNumber", r."LastUserId", r."ProcessCode", r."Updated", r."Year"  
      FROM "RequestNumberCounter" AS r
      WHERE r."ProcessCode" = @__processCode_0 AND r."Year" = @__year_1
      LIMIT 1
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (4ms) [Parameters=[@__p_0='cd015b3b-91e9-4f5f-853f-3f3a6884dcf1' (Nullable = true)], CommandType='Text', CommandTimeout='30']
      SELECT r."Id", r."CreateUserId", r."Created", r."LastNumber", r."LastUserId", r."ProcessCode", r."Updated", r."Year"  
      FROM "RequestNumberCounter" AS r
      WHERE r."Id" = @__p_0
      LIMIT 1
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (15ms) [Parameters=[@p0='2025-08-04T14:21:22.0645470' (DbType = DateTime), @p1='cd015b3b-91e9-4f5f-
853f-3f3a6884dcf1', @p2='2025-08-07T18:14:23.8301480+05:00' (DbType = DateTime), @p3='{"EntityId":"cd015b3b-91e9-4f5f-853f-3
f3a6884dcf1","UserName":"","EventDateTime":"2025-08-07T18:14:23.830148+05:00","EventName":"RequestNumberCounterChangedEvent"
}' (Nullable = false), @p4='RequestNumberCounterChangedEvent' (Nullable = false), @p5='False', @p6='' (Nullable = false), @p8='cd015b3b-91e9-4f5f-853f-3f3a6884dcf1', @p7='29'], CommandType='Text', CommandTimeout='30']
      INSERT INTO "JournalEvents" ("EntityCreated", "EntityId", "EventDateTime", "EventJson", "EventName", "IsSentToQueue", "UserName")
      VALUES (@p0, @p1, @p2, @p3, @p4, @p5, @p6)
      RETURNING "Id";
      UPDATE "RequestNumberCounter" SET "LastNumber" = @p7
      WHERE "Id" = @p8;
info: System.Net.Http.HttpClient.Refit.Implementation.Generated+BpmBaseApiServicesCamundaServiceCamundaICamundaProvider, BpmBaseApi.Services, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null.LogicalHandler[100]
      Start processing HTTP request POST http://172.31.120.61:8090/processes/start?processId=Memo
info: System.Net.Http.HttpClient.Refit.Implementation.Generated+BpmBaseApiServicesCamundaServiceCamundaICamundaProvider, BpmBaseApi.Services, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null.ClientHandler[100]
      Sending HTTP request POST http://172.31.120.61:8090/processes/start?processId=Memo
info: System.Net.Http.HttpClient.Refit.Implementation.Generated+BpmBaseApiServicesCamundaServiceCamundaICamundaProvider, BpmBaseApi.Services, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null.ClientHandler[101]
      Received HTTP response headers after 331.2215ms - 200
info: System.Net.Http.HttpClient.Refit.Implementation.Generated+BpmBaseApiServicesCamundaServiceCamundaICamundaProvider, BpmBaseApi.Services, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null.LogicalHandler[101]
      End processing HTTP request after 392.951ms - 200
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (4ms) [Parameters=[@__p_0='00000000-0000-0000-0000-000000000000' (Nullable = true)], CommandType='Text', CommandTimeout='30']
      SELECT p."Id", p."BlockCode", p."BlockName", p."CreateUserId", p."Created", p."InitiatorCode", p."InitiatorName", p."L
astUserId", p."PayloadJson", p."ProcessCode", p."ProcessId", p."ProcessInstanceId", p."ProcessName", p."RegNumber", p."StatusCode", p."StatusName", p."Title", p."Updated"
      FROM "ProcessData" AS p
      WHERE p."Id" = @__p_0
      LIMIT 1
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (5ms) [Parameters=[@p0='2025-08-07T18:14:26.0585844+05:00' (DbType = DateTime), @p1='736baa41-e8d9-
4522-81b7-3513c42fe162', @p2='2025-08-07T18:14:25.9669514+05:00' (DbType = DateTime), @p3='{"EntityId":"736baa41-e8d9-4522-8
1b7-3513c42fe162","UserName":"","EventDateTime":"2025-08-07T18:14:25.9669514+05:00","EventName":"ProcessDataCreatedEvent"}' 
(Nullable = false), @p4='ProcessDataCreatedEvent' (Nullable = false), @p5='False', @p6='' (Nullable = false), @p7='736baa41-
e8d9-4522-81b7-3513c42fe162', @p8=NULL, @p9=NULL, @p10='' (Nullable = false), @p11='2025-08-07T18:14:26.0585844+05:00' (DbTy
pe = DateTime), @p12='b.shymkentbay' (Nullable = false), @p13='Шымкентбай Ба?ытжан Бахтияр?лы' (Nullable = false), @p14='' (
Nullable = false), @p15='{"regData":{"userCode":"b.shymkentbay","userName":"Шымкентбай Ба?ытжан Бахтияр?лы","departmentId":"
19.100512","departmentName":"Управление разработки пенсионного учета","startdate":"2025-08-05T10:09:46.584Z","regnum":""},"s
ysInfo":{"userCode":"b.shymkentbay","userName":"Шымкентбай Ба?ытжан Бахтияр?лы","comment":"comment","action":"submit","condi
tion":"string"},"initiator":{"id":7820,"name":"Шымкентбай Ба?ытжан Бахтияр?лы","position":"Главный специалист","login":"b.sh
ymkentbay","statusCode":6,"statusDescription":"Работа","depId":"19.100512","depName":"Управление разработки пенсионного учет
а","parentDepId":"19.100500","parentDepName":"Департамент цифровизации","isFilial":false,"mail":"b.shymkentbay@enpf.kz","loc
alPhone":"0","mobilePhone":"+7(708) 927-44-98","isManager":false,"managerTabNumber":"4340","disabled":false,"tabNumber":"00З
П-00292"},"approvers":[{"loginAD":"m.ilespayev","id":611,"name":"Илеспаев Меииржан Анварович","shortName":null,"position":"З
аместитель директора департамента","login":"m.ilespayev","statusCode":6,"statusDescription":"Работа","depId":"19.100500","de
pName":"Департамент цифровизации","parentDepId":"19.100500","parentDepName":"Департамент цифровизации","isFilial":false,"mai
l":"m.ilespayev@enpf.kz","localPhone":"0","mobilePhone":"+7(702) 171-71-14","isManager":true,"managerTabNumber":"4303","disa
bled":false,"tabNumber":"00ЗП-00240"},{"loginAD":"a.ysmail","id":1545,"name":"Ысмаил Ар?ынбек Байдабек?лы","shortName":null,
"position":"Главный специалист","login":"a.ysmail","statusCode":6,"statusDescription":"Работа","depId":"19.100508","depName"
:"Управление разработки фронтальных систем","parentDepId":"19.100500","parentDepName":"Департамент цифровизации","isFilial":
false,"mail":"a.ysmail@enpf.kz","localPhone":"0","mobilePhone":"+7(702) 778-53-30","isManager":false,"managerTabNumber":"434
0","disabled":false,"tabNumber":"00ЗП-00289"}],"recipients":[{"loginAD":"l.iskender","id":633,"name":"Искендер Лесхан Мурат?
лы","shortName":null,"position":"Главный специалист","login":"l.iskender","statusCode":6,"statusDescription":"Работа","depId
":"19.100509","depName":"Управление разработки Web приложений и сервисов","parentDepId":"19.100500","parentDepName":"Департа
мент цифровизации","isFilial":false,"mail":"l.iskender@enpf.kz","localPhone":"0","mobilePhone":"+7(707) 517-04-67","isManage
r":false,"managerTabNumber":"4340","disabled":false,"tabNumber":"00ЗП-00083"},{"loginAD":"i.dosgali","id":414,"name":"Досгал
и Искандер Досгали?лы","shortName":null,"position":"Главный специалист","login":"i.dosgali","statusCode":6,"statusDescriptio
n":"Работа","depId":"19.100509","depName":"Управление разработки Web приложений и сервисов","parentDepId":"19.100500","paren
tDepName":"Департамент цифровизации","isFilial":false,"mail":"i.dosgali@enpf.kz","localPhone":"0","mobilePhone":"+7(747) 790
-29-49","isManager":false,"managerTabNumber":"4340","disabled":false,"tabNumber":"00ЗП-00275"}],"signer":{"loginAD":"a.arist
ombekov","id":168,"name":"Аристомбеков Арстан Рамазанулы","shortName":null,"position":"Директор департамента","login":"a.ari
stombekov","statusCode":5,"statusDescription":"Отпуск основной","depId":"19.100500","depName":"Департамент цифровизации","pa
rentDepId":"19.100500","parentDepName":"Департамент цифровизации","isFilial":false,"mail":"a.aristombekov@enpf.kz","localPho
ne":"0","mobilePhone":"+7(705) 950-90-65","isManager":true,"managerTabNumber":"4303","disabled":false,"tabNumber":"4340"},"p
rocessData":{"documentLang":"","nomenclatureId":"1","documentTitle":"тема документа","grif":"string","pageCount":"5","signTy
pe":"string","allGroup":"string","meGroup":"string","documentBody":"<p><strong>ТЕКС&nbsp;СЗ</strong></p>"},"files":[{"fileId
":"0ec544b8-6d84-4fd5-93fc-ff22f13053b3","fileName":"Рисковое событие.docx","fileType":"scan"},{"fileId":"8d1b2532-b727-484d
-877f-1e7658851a2e","fileName":"ffh.pdf","fileType":"scan"}]}' (Nullable = false), @p16='Memo' (Nullable = false), @p17='662
9d7a0-98ef-400e-87e7-8ff357adcffb', @p18='7a9adece-7390-11f0-9e4e-0242ac160004', @p19='Служебная записка' (Nullable = false)
, @p20='ME-0029-2025' (Nullable = false), @p21='Started' (Nullable = false), @p22='В работе' (Nullable = false), @p23='тема документа', @p24='0001-01-01T00:00:00.0000000' (DbType = DateTime)], CommandType='Text', CommandTimeout='30']
      INSERT INTO "JournalEvents" ("EntityCreated", "EntityId", "EventDateTime", "EventJson", "EventName", "IsSentToQueue", "UserName")
      VALUES (@p0, @p1, @p2, @p3, @p4, @p5, @p6)
      RETURNING "Id";
      INSERT INTO "ProcessData" ("Id", "BlockCode", "BlockName", "CreateUserId", "Created", "InitiatorCode", "InitiatorName"
, "LastUserId", "PayloadJson", "ProcessCode", "ProcessId", "ProcessInstanceId", "ProcessName", "RegNumber", "StatusCode", "StatusName", "Title", "Updated")
      VALUES (@p7, @p8, @p9, @p10, @p11, @p12, @p13, @p14, @p15, @p16, @p17, @p18, @p19, @p20, @p21, @p22, @p23, @p24);     
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (3ms) [Parameters=[@__p_0='6a8c295d-b1b9-4675-b0f0-eb9710b7b7dc' (Nullable = true)], CommandType='Text', CommandTimeout='30']
      SELECT p."Id", p."CreateUserId", p."Created", p."FileName", p."FileType", p."LastUserId", p."ProcessDataId", p."Updated"
      FROM "ProcessFile" AS p
      WHERE p."Id" = @__p_0
      LIMIT 1
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (9ms) [Parameters=[@p0='2025-08-07T18:14:26.4902961+05:00' (DbType = DateTime), @p1='6a8c295d-b1b9-
4675-b0f0-eb9710b7b7dc', @p2='2025-08-07T18:14:26.4461952+05:00' (DbType = DateTime), @p3='{"EntityId":"6a8c295d-b1b9-4675-b
0f0-eb9710b7b7dc","UserName":"","EventDateTime":"2025-08-07T18:14:26.4461952+05:00","EventName":"ProcessFileCreatedEvent"}' 
(Nullable = false), @p4='ProcessFileCreatedEvent' (Nullable = false), @p5='False', @p6='' (Nullable = false), @p7='019884ab-
294f-77ae-ae7f-9d655eab103d', @p8='' (Nullable = false), @p9='2025-08-07T18:14:26.4902961+05:00' (DbType = DateTime), @p10=N
ULL (Nullable = false), @p11=NULL (Nullable = false), @p12='' (Nullable = false), @p13='00000000-0000-0000-0000-000000000000', @p14='0001-01-01T00:00:00.0000000' (DbType = DateTime)], CommandType='Text', CommandTimeout='30']
      INSERT INTO "JournalEvents" ("EntityCreated", "EntityId", "EventDateTime", "EventJson", "EventName", "IsSentToQueue", "UserName")
      VALUES (@p0, @p1, @p2, @p3, @p4, @p5, @p6)
      RETURNING "Id";
      INSERT INTO "ProcessFile" ("Id", "CreateUserId", "Created", "FileName", "FileType", "LastUserId", "ProcessDataId", "Updated")
      VALUES (@p7, @p8, @p9, @p10, @p11, @p12, @p13, @p14);
fail: Microsoft.EntityFrameworkCore.Update[10000]
      An exception occurred in the database while saving changes for context type 'BpmBaseApi.Persistence.ApplicationDbContext'.
      Microsoft.EntityFrameworkCore.DbUpdateException: An error occurred while saving the entity changes. See the inner exception for details.
       ---> Npgsql.PostgresException (0x80004005): 23502: значение NULL в столбце "FileName" отношения "ProcessFile" нарушает ограничение NOT NULL

      DETAIL: Detail redacted as it may contain sensitive data. Specify 'Include Error Detail' in the connection string to include this information.
         at Npgsql.Internal.NpgsqlConnector.ReadMessageLong(Boolean async, DataRowLoadingMode dataRowLoadingMode, Boolean readingNotifications, Boolean isReadingPrependedMessage)
         at System.Runtime.CompilerServices.PoolingAsyncValueTaskMethodBuilder`1.StateMachineBox`1.System.Threading.Tasks.Sources.IValueTaskSource<TResult>.GetResult(Int16 token)
         at Npgsql.NpgsqlDataReader.NextResult(Boolean async, Boolean isConsuming, CancellationToken cancellationToken)     
         at Npgsql.NpgsqlDataReader.NextResult(Boolean async, Boolean isConsuming, CancellationToken cancellationToken)     
         at Npgsql.EntityFrameworkCore.PostgreSQL.Update.Internal.NpgsqlModificationCommandBatch.Consume(RelationalDataReader reader, Boolean async, CancellationToken cancellationToken)
        Exception data:
          Severity: ОШИБКА
          SqlState: 23502
          MessageText: значение NULL в столбце "FileName" отношения "ProcessFile" нарушает ограничение NOT NULL
          Detail: Detail redacted as it may contain sensitive data. Specify 'Include Error Detail' in the connection string to include this information.
          SchemaName: public
          TableName: ProcessFile
          ColumnName: FileName
          File: execMain.c
          Line: 1988
          Routine: ExecConstraints
         --- End of inner exception stack trace ---
         at Npgsql.EntityFrameworkCore.PostgreSQL.Update.Internal.NpgsqlModificationCommandBatch.Consume(RelationalDataReader reader, Boolean async, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.Update.ReaderModificationCommandBatch.ExecuteAsync(IRelationalConnection connection, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.Update.ReaderModificationCommandBatch.ExecuteAsync(IRelationalConnection connection, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.Update.Internal.BatchExecutor.ExecuteAsync(IEnumerable`1 commandBatches, IRelationalConnection connection, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.Update.Internal.BatchExecutor.ExecuteAsync(IEnumerable`1 commandBatches, IRelationalConnection connection, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.Update.Internal.BatchExecutor.ExecuteAsync(IEnumerable`1 commandBatches, IRelationalConnection connection, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.Storage.RelationalDatabase.SaveChangesAsync(IList`1 entries, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.StateManager.SaveChangesAsync(IList`1 entriesToSave, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.StateManager.SaveChangesAsync(StateManager stateManager, Boolean acceptAllChangesOnSuccess, CancellationToken cancellationToken)
         at Npgsql.EntityFrameworkCore.PostgreSQL.Storage.Internal.NpgsqlExecutionStrategy.ExecuteAsync[TState,TResult](TState state, Func`4 operation, Func`4 verifySucceeded, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.DbContext.SaveChangesAsync(Boolean acceptAllChangesOnSuccess, CancellationToken cancellationToken)
      Microsoft.EntityFrameworkCore.DbUpdateException: An error occurred while saving the entity changes. See the inner exception for details.
       ---> Npgsql.PostgresException (0x80004005): 23502: значение NULL в столбце "FileName" отношения "ProcessFile" нарушает ограничение NOT NULL

      DETAIL: Detail redacted as it may contain sensitive data. Specify 'Include Error Detail' in the connection string to include this information.
         at Npgsql.Internal.NpgsqlConnector.ReadMessageLong(Boolean async, DataRowLoadingMode dataRowLoadingMode, Boolean readingNotifications, Boolean isReadingPrependedMessage)
         at System.Runtime.CompilerServices.PoolingAsyncValueTaskMethodBuilder`1.StateMachineBox`1.System.Threading.Tasks.Sources.IValueTaskSource<TResult>.GetResult(Int16 token)
         at Npgsql.NpgsqlDataReader.NextResult(Boolean async, Boolean isConsuming, CancellationToken cancellationToken)     
         at Npgsql.NpgsqlDataReader.NextResult(Boolean async, Boolean isConsuming, CancellationToken cancellationToken)     
         at Npgsql.EntityFrameworkCore.PostgreSQL.Update.Internal.NpgsqlModificationCommandBatch.Consume(RelationalDataReader reader, Boolean async, CancellationToken cancellationToken)
        Exception data:
          Severity: ОШИБКА
          SqlState: 23502
          MessageText: значение NULL в столбце "FileName" отношения "ProcessFile" нарушает ограничение NOT NULL
          Detail: Detail redacted as it may contain sensitive data. Specify 'Include Error Detail' in the connection string to include this information.
          SchemaName: public
          TableName: ProcessFile
          ColumnName: FileName
          File: execMain.c
          Line: 1988
          Routine: ExecConstraints
         --- End of inner exception stack trace ---
         at Npgsql.EntityFrameworkCore.PostgreSQL.Update.Internal.NpgsqlModificationCommandBatch.Consume(RelationalDataReader reader, Boolean async, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.Update.ReaderModificationCommandBatch.ExecuteAsync(IRelationalConnection connection, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.Update.ReaderModificationCommandBatch.ExecuteAsync(IRelationalConnection connection, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.Update.Internal.BatchExecutor.ExecuteAsync(IEnumerable`1 commandBatches, IRelationalConnection connection, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.Update.Internal.BatchExecutor.ExecuteAsync(IEnumerable`1 commandBatches, IRelationalConnection connection, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.Update.Internal.BatchExecutor.ExecuteAsync(IEnumerable`1 commandBatches, IRelationalConnection connection, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.Storage.RelationalDatabase.SaveChangesAsync(IList`1 entries, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.StateManager.SaveChangesAsync(IList`1 entriesToSave, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.StateManager.SaveChangesAsync(StateManager stateManager, Boolean acceptAllChangesOnSuccess, CancellationToken cancellationToken)
         at Npgsql.EntityFrameworkCore.PostgreSQL.Storage.Internal.NpgsqlExecutionStrategy.ExecuteAsync[TState,TResult](TState state, Func`4 operation, Func`4 verifySucceeded, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.DbContext.SaveChangesAsync(Boolean acceptAllChangesOnSuccess, CancellationToken cancellationToken)
fail: Microsoft.AspNetCore.Diagnostics.DeveloperExceptionPageMiddleware[1]
      An unhandled exception has occurred while executing the request.
      Microsoft.EntityFrameworkCore.DbUpdateException: An error occurred while saving the entity changes. See the inner exception for details.
       ---> Npgsql.PostgresException (0x80004005): 23502: значение NULL в столбце "FileName" отношения "ProcessFile" нарушает ограничение NOT NULL

      DETAIL: Detail redacted as it may contain sensitive data. Specify 'Include Error Detail' in the connection string to include this information.
         at Npgsql.Internal.NpgsqlConnector.ReadMessageLong(Boolean async, DataRowLoadingMode dataRowLoadingMode, Boolean readingNotifications, Boolean isReadingPrependedMessage)
         at System.Runtime.CompilerServices.PoolingAsyncValueTaskMethodBuilder`1.StateMachineBox`1.System.Threading.Tasks.Sources.IValueTaskSource<TResult>.GetResult(Int16 token)
         at Npgsql.NpgsqlDataReader.NextResult(Boolean async, Boolean isConsuming, CancellationToken cancellationToken)     
         at Npgsql.NpgsqlDataReader.NextResult(Boolean async, Boolean isConsuming, CancellationToken cancellationToken)     
         at Npgsql.EntityFrameworkCore.PostgreSQL.Update.Internal.NpgsqlModificationCommandBatch.Consume(RelationalDataReader reader, Boolean async, CancellationToken cancellationToken)
        Exception data:
          Severity: ОШИБКА
          SqlState: 23502
          MessageText: значение NULL в столбце "FileName" отношения "ProcessFile" нарушает ограничение NOT NULL
          Detail: Detail redacted as it may contain sensitive data. Specify 'Include Error Detail' in the connection string to include this information.
          SchemaName: public
          TableName: ProcessFile
          ColumnName: FileName
          File: execMain.c
          Line: 1988
          Routine: ExecConstraints
         --- End of inner exception stack trace ---
         at Npgsql.EntityFrameworkCore.PostgreSQL.Update.Internal.NpgsqlModificationCommandBatch.Consume(RelationalDataReader reader, Boolean async, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.Update.ReaderModificationCommandBatch.ExecuteAsync(IRelationalConnection connection, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.Update.ReaderModificationCommandBatch.ExecuteAsync(IRelationalConnection connection, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.Update.Internal.BatchExecutor.ExecuteAsync(IEnumerable`1 commandBatches, IRelationalConnection connection, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.Update.Internal.BatchExecutor.ExecuteAsync(IEnumerable`1 commandBatches, IRelationalConnection connection, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.Update.Internal.BatchExecutor.ExecuteAsync(IEnumerable`1 commandBatches, IRelationalConnection connection, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.Storage.RelationalDatabase.SaveChangesAsync(IList`1 entries, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.StateManager.SaveChangesAsync(IList`1 entriesToSave, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.StateManager.SaveChangesAsync(StateManager stateManager, Boolean acceptAllChangesOnSuccess, CancellationToken cancellationToken)
         at Npgsql.EntityFrameworkCore.PostgreSQL.Storage.Internal.NpgsqlExecutionStrategy.ExecuteAsync[TState,TResult](TState state, Func`4 operation, Func`4 verifySucceeded, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.DbContext.SaveChangesAsync(Boolean acceptAllChangesOnSuccess, CancellationToken cancellationToken)
         at Microsoft.EntityFrameworkCore.DbContext.SaveChangesAsync(Boolean acceptAllChangesOnSuccess, CancellationToken cancellationToken)
         at BpmBaseApi.Persistence.Repositories.JournaledGenericRepository`1.RaiseEvent(BaseEntityEvent event, CancellationT
oken cancellationToken, Boolean autoCommit) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi.Persistence\Repositories\JournaledGenericRepository.cs:line 124
         at BpmBaseApi.Application.CommandHandlers.Process.StartProcessCommandHandler.Handle(StartProcessCommand command, Ca
ncellationToken cancellationToken) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi.Application\CommandHandlers\Process\StartProcessCommandHandler.cs:line 98
         at BpmBaseApi.Controllers.V1.ProcessController.StartProcessAsync(StartProcessCommand command, CancellationToken cancellationToken) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi\Controllers\V1\ProcessController.cs:line 55
         at Microsoft.AspNetCore.Mvc.Infrastructure.ActionMethodExecutor.TaskOfIActionResultExecutor.Execute(ActionContext actionContext, IActionResultTypeMapper mapper, ObjectMethodExecutor executor, Object controller, Object[] arguments)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.<InvokeActionMethodAsync>g__Logged|12_1(ControllerActionInvoker invoker)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.<InvokeNextActionFilterAsync>g__Awaited|10_0(ControllerActionInvoker invoker, Task lastTask, State next, Scope scope, Object state, Boolean isCompleted)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.Rethrow(ActionExecutedContextSealed context)    
         at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.Next(State& next, Scope& scope, Object& state, Boolean& isCompleted)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.<InvokeInnerFilterAsync>g__Awaited|13_0(ControllerActionInvoker invoker, Task lastTask, State next, Scope scope, Object state, Boolean isCompleted)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeFilterPipelineAsync>g__Awaited|20_0(ResourceInvoker invoker, Task lastTask, State next, Scope scope, Object state, Boolean isCompleted)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeAsync>g__Logged|17_1(ResourceInvoker invoker)    
         at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeAsync>g__Logged|17_1(ResourceInvoker invoker)    
         at BpmBaseApi.Middlewares.ErrorHandlerMiddleware.Invoke(HttpContext context) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi\Middlewares\ErrorHandlerMiddleware.cs:line 24
         at BpmBaseApi.Middlewares.DefineUserHandler.Invoke(HttpContext context) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi\Middlewares\DefineUserHandler.cs:line 30
         at BpmBaseApi.Middlewares.CorrelationMiddleware.Invoke(HttpContext context) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi\Middlewares\CorrelationMiddleware.cs:line 16
         at Microsoft.AspNetCore.Authorization.AuthorizationMiddleware.Invoke(HttpContext context)
         at Microsoft.AspNetCore.Authentication.AuthenticationMiddleware.Invoke(HttpContext context)
         at Swashbuckle.AspNetCore.ReDoc.ReDocMiddleware.Invoke(HttpContext httpContext)
         at Swashbuckle.AspNetCore.SwaggerUI.SwaggerUIMiddleware.Invoke(HttpContext httpContext)
         at Swashbuckle.AspNetCore.Swagger.SwaggerMiddleware.Invoke(HttpContext httpContext, ISwaggerProvider swaggerProvider)
         at Microsoft.AspNetCore.Diagnostics.DeveloperExceptionPageMiddlewareImpl.Invoke(HttpContext context)


