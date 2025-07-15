info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (355ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      ALTER TABLE public."Employees"
            ALTER COLUMN status_code
            TYPE integer
            USING status_code::integer;
fail: Microsoft.EntityFrameworkCore.Database.Command[20102]
      Failed executing DbCommand (52ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      ALTER TABLE public."Employees" ALTER COLUMN id DROP IDENTITY;
      ALTER TABLE public."Employees" ALTER COLUMN id SET DEFAULT (nextval('public.employees_id_seq'::regclass));
      COMMENT ON COLUMN public."Employees".id IS NULL;
Failed executing DbCommand (52ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
ALTER TABLE public."Employees" ALTER COLUMN id DROP IDENTITY;
ALTER TABLE public."Employees" ALTER COLUMN id SET DEFAULT (nextval('public.employees_id_seq'::regclass));
COMMENT ON COLUMN public."Employees".id IS NULL;
Npgsql.PostgresException (0x80004005): 42P01: отношение "public.employees_id_seq" не существует
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
    MessageText: отношение "public.employees_id_seq" не существует
    File: regproc.c
    Line: 914
    Routine: regclassin
42P01: отношение "public.employees_id_seq" не существует
