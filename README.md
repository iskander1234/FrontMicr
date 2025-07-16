PS C:\BPM\bpm\bpmbaseapi> dotnet ef migrations remove --project BpmBaseApi.Persistence --startup-project BpmBaseApi
Build started...
Build succeeded.
warn: Microsoft.EntityFrameworkCore.Model.Validation[10400]
      Sensitive data logging is enabled. Log entries and exception messages may include sensitive application data; this mode should only be enabled during development.
fail: Microsoft.EntityFrameworkCore.Database.Connection[20004]
      An error occurred using the connection to database 'bpmbase6Userid=postgres' on server 'tcp://localhost:5432'.
An error occurred using the connection to database 'bpmbase6Userid=postgres' on server 'tcp://localhost:5432'.
Npgsql.PostgresException (0x80004005): 28P01: ???????????? "i.dosgali" ?? ?????? ???????? ??????????? (?? ??????)
   at Npgsql.Internal.NpgsqlConnector.ReadMessageLong(Boolean async, DataRowLoadingMode dataRowLoadingMode, Boolean readingNotifications, Boolean isReadingPrependedMessage)
   at System.Runtime.CompilerServices.PoolingAsyncValueTaskMethodBuilder`1.StateMachineBox`1.System.Threading.Tasks.Sources.IValueTaskSource<TResult>.GetResult(Int16 token)
   at Npgsql.Internal.NpgsqlConnector.AuthenticateSASL(List`1 mechanisms, String username, Boolean async, CancellationToken cancellationToken)
   at Npgsql.Internal.NpgsqlConnector.Authenticate(String username, NpgsqlTimeout timeout, Boolean async, CancellationToken cancellationToken)
   at Npgsql.Internal.NpgsqlConnector.<Open>g__OpenCore|214_1(NpgsqlConnector conn, SslMode sslMode, NpgsqlTimeout timeout, Boolean async, CancellationToken cancellationToken)
   at Npgsql.Internal.NpgsqlConnector.Open(NpgsqlTimeout timeout, Boolean async, CancellationToken cancellationToken)
   at Npgsql.PoolingDataSource.OpenNewConnector(NpgsqlConnection conn, NpgsqlTimeout timeout, Boolean async, CancellationToken cancellationToken)
   at Npgsql.PoolingDataSource.<Get>g__RentAsync|33_0(NpgsqlConnection conn, NpgsqlTimeout timeout, Boolean async, CancellationToken cancellationToken)
   at Npgsql.NpgsqlConnection.<Open>g__OpenAsync|42_0(Boolean async, CancellationToken cancellationToken)
   at Npgsql.NpgsqlConnection.Open()
   at Microsoft.EntityFrameworkCore.Storage.RelationalConnection.OpenDbConnection(Boolean errorsExpected)
   at Microsoft.EntityFrameworkCore.Storage.RelationalConnection.OpenInternal(Boolean errorsExpected)
   at Microsoft.EntityFrameworkCore.Storage.RelationalConnection.Open(Boolean errorsExpected)
   at Microsoft.EntityFrameworkCore.Storage.RelationalCommand.ExecuteReader(RelationalCommandParameterObject parameterObject)
   at Microsoft.EntityFrameworkCore.Migrations.HistoryRepository.GetAppliedMigrations()
   at Npgsql.EntityFrameworkCore.PostgreSQL.Migrations.Internal.NpgsqlHistoryRepository.GetAppliedMigrations()
   at Microsoft.EntityFrameworkCore.Migrations.Design.MigrationsScaffolder.RemoveMigration(String projectDir, String rootNamespace, Boolean force, String language, Boolean dryRun)
   at Microsoft.EntityFrameworkCore.Design.Internal.MigrationsOperations.RemoveMigration(String contextType, Boolean force, Boolean dryRun)
   at Microsoft.EntityFrameworkCore.Design.OperationExecutor.RemoveMigrationImpl(String contextType, Boolean force, Boolean dryRun)
   at Microsoft.EntityFrameworkCore.Design.OperationExecutor.RemoveMigration.<>c__DisplayClass0_0.<.ctor>b__0()
   at Microsoft.EntityFrameworkCore.Design.OperationExecutor.OperationBase.<>c__DisplayClass3_0`1.<Execute>b__0()
   at Microsoft.EntityFrameworkCore.Design.OperationExecutor.OperationBase.Execute(Action action)
  Exception data:
    Severity: ?????
    SqlState: 28P01
    MessageText: ???????????? "i.dosgali" ?? ?????? ???????? ??????????? (?? ??????)
    File: auth.c
    Line: 324
    Routine: auth_failed
28P01: ???????????? "i.dosgali" ?? ?????? ???????? ??????????? (?? ??????)
