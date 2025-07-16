      Applying migration '20250716053806_FixedTableEmployeesAndDepatments'.
Applying migration '20250716053806_FixedTableEmployeesAndDepatments'.
fail: Microsoft.EntityFrameworkCore.Database.Command[20102]
      Failed executing DbCommand (57ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      ALTER TABLE "Employees" DROP CONSTRAINT "PK_Employees";
Failed executing DbCommand (57ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
ALTER TABLE "Employees" DROP CONSTRAINT "PK_Employees";
Npgsql.PostgresException (0x80004005): 42P01: отношение "Employees" не существует
   at Npgsql.Internal.NpgsqlConnector.ReadMessageLong(Boolean async, DataRowLoadingMode dataRowLoadingMode, Boolean readingNotifications, Boolean isReadingPrependedMessage)
   at System.Runtime.CompilerServices.PoolingAsyncValueTaskMethodBuilder`1.StateMachineBox`1.System.Threading.Tasks.Sources.IValueTaskSource<TResult>.GetResult(Int16 token)
   at Npgsql.NpgsqlDataReader.NextResult(Boolean async, Boolean isConsuming, CancellationToken cancellationToken)
   at Npgsql.NpgsqlDataReader.NextResult(Boolean async, Boolean isConsuming, CancellationToken cancellationToken)
   at Npgsql.NpgsqlDataReader.NextResult()
   at Npgsql.NpgsqlCommand.ExecuteReader(Boolean async, CommandBehavior behavior, CancellationToken cancellationToken)
   at Npgsql.NpgsqlCommand.ExecuteReader(Boolean async, CommandBehavior behavior, CancellationToken cancellationToken)
   at Npgsql.NpgsqlCommand.ExecuteNonQuery(Boolean async, CancellationToken cancellationToken)
   at Npgsql.NpgsqlCommand.ExecuteNonQuery()
   at Microsoft.EntityFrameworkCore.Storage.RelationalCommand.ExecuteNonQuery(RelationalCommandParameterObject parameterObject)
   at Microsoft.EntityFrameworkCore.Migrations.MigrationCommand.ExecuteNonQuery(IRelationalConnection connection, IReadOnlyDictionary`2 parameterValues)
   at Microsoft.EntityFrameworkCore.Migrations.Internal.MigrationCommandExecutor.Execute(IReadOnlyList`1 migrationCommands, IRelationalConnection connection, MigrationExecutionState executionState, Boolean beginTransaction, Boolean commitTransaction, Nullable`1 isolationLevel)
   at Microsoft.EntityFrameworkCore.Migrations.Internal.MigrationCommandExecutor.<>c.<ExecuteNonQuery>b__3_1(DbContext _, ValueTuple`6 s)
   at Npgsql.EntityFrameworkCore.PostgreSQL.Storage.Internal.NpgsqlExecutionStrategy.Execute[TState,TResult](TState state, Func`3 operation, Func`3 verifySucceeded)
   at Microsoft.EntityFrameworkCore.Migrations.Internal.MigrationCommandExecutor.ExecuteNonQuery(IReadOnlyList`1 migrationCommands, IRelationalConnection connection, MigrationExecutionState executionState, Boolean commitTransaction, Nullable`1 isolationLevel)
   at Microsoft.EntityFrameworkCore.Migrations.Internal.Migrator.MigrateImplementation(DbContext context, String targetMigration, MigrationExecutionState state, Boolean useTransaction)    
   at Microsoft.EntityFrameworkCore.Migrations.Internal.Migrator.<>c.<Migrate>b__20_1(DbContext c, ValueTuple`4 s)
   at Npgsql.EntityFrameworkCore.PostgreSQL.Storage.Internal.NpgsqlExecutionStrategy.Execute[TState,TResult](TState state, Func`3 operation, Func`3 verifySucceeded)
   at Microsoft.EntityFrameworkCore.Migrations.Internal.Migrator.Migrate(String targetMigration)
   at Npgsql.EntityFrameworkCore.PostgreSQL.Migrations.Internal.NpgsqlMigrator.Migrate(String targetMigration)
   at Microsoft.EntityFrameworkCore.Design.Internal.MigrationsOperations.UpdateDatabase(String targetMigration, String connectionString, String contextType)
   at Microsoft.EntityFrameworkCore.Design.OperationExecutor.UpdateDatabaseImpl(String targetMigration, String connectionString, String contextType)
   at Microsoft.EntityFrameworkCore.Design.OperationExecutor.UpdateDatabase.<>c__DisplayClass0_0.<.ctor>b__0()
   at Microsoft.EntityFrameworkCore.Design.OperationExecutor.OperationBase.Execute(Action action)
  Exception data:
    Severity: ОШИБКА
    SqlState: 42P01
    MessageText: отношение "Employees" не существует
    File: namespace.c
    Line: 639
    Routine: RangeVarGetRelidExtended
42P01: отношение "Employees" не существует
PS C:\BPM\bpm\bpmbaseapi> dotnet ef database update --project BpmBaseApi.Persistence --startup-project BpmBaseApi
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
   at Npgsql.EntityFrameworkCore.PostgreSQL.Migrations.Internal.NpgsqlMigrator.Migrate(String targetMigration)
   at Microsoft.EntityFrameworkCore.Design.Internal.MigrationsOperations.UpdateDatabase(String targetMigration, String connectionString, String contextType)
   at Microsoft.EntityFrameworkCore.Design.OperationExecutor.UpdateDatabaseImpl(String targetMigration, String connectionString, String contextType)
   at Microsoft.EntityFrameworkCore.Design.OperationExecutor.UpdateDatabase.<>c__DisplayClass0_0.<.ctor>b__0()
   at Microsoft.EntityFrameworkCore.Design.OperationExecutor.OperationBase.Execute(Action action)
  Exception data:
    Severity: ?????
    SqlState: 28P01
    MessageText: ???????????? "i.dosgali" ?? ?????? ???????? ??????????? (?? ??????)
    File: auth.c
    Line: 324
    Routine: auth_failed
28P01: ???????????? "i.dosgali" ?? ?????? ???????? ??????????? (?? ??????)
