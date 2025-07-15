info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (1ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      ALTER TABLE public."Departments" RENAME COLUMN parent_code TO parent_id;
fail: Microsoft.EntityFrameworkCore.Database.Command[20102]
      Failed executing DbCommand (119ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      ALTER TABLE public."Departments" ADD CONSTRAINT departments2_parent_id_fkey FOREIGN KEY (parent_id) REFERENCES public."Departments" (id) ON DELETE SET NULL;
Failed executing DbCommand (119ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
ALTER TABLE public."Departments" ADD CONSTRAINT departments2_parent_id_fkey FOREIGN KEY (parent_id) REFERENCES public."Departments" (id) ON DELETE SET NULL;
Npgsql.PostgresException (0x80004005): 23503: INSERT или UPDATE в таблице "Departments" нарушает ограничение внешнего ключа "departments2_parent_id_fkey"

DETAIL: Detail redacted as it may contain sensitive data. Specify 'Include Error Detail' in the connection string to include this information.
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
    SqlState: 23503
    MessageText: INSERT или UPDATE в таблице "Departments" нарушает ограничение внешнего ключа "departments2_parent_id_fkey"
    Detail: Detail redacted as it may contain sensitive data. Specify 'Include Error Detail' in the connection string to include this information.
    SchemaName: public
    TableName: Departments
    ConstraintName: departments2_parent_id_fkey
    File: ri_triggers.c
    Line: 2610
    Routine: ri_ReportViolation
23503: INSERT или UPDATE в таблице "Departments" нарушает ограничение внешнего ключа "departments2_parent_id_fkey"

DETAIL: Detail redacted as it may contain sensitive data. Specify 'Include Error Detail' in the connection string to include this information.

using Microsoft.EntityFrameworkCore.Migrations;

#nullable disable

namespace BpmBaseApi.Persistence.Migrations
{
    public partial class FixedTableEmployeesAndDepatments : Migration
    {
        protected override void Up(MigrationBuilder migrationBuilder)
        {
            // 1. Удаляем старые PK
            migrationBuilder.DropPrimaryKey(
                name: "PK_Employees",
                table: "Employees");

            migrationBuilder.DropPrimaryKey(
                name: "PK_departments",
                table: "Departments");

            // 2. Удаляем sequence, если была
            migrationBuilder.Sql(@"DROP SEQUENCE IF EXISTS public.departments_id_seq CASCADE;");

            // 3. Перемещаем таблицы в schema public
            migrationBuilder.EnsureSchema(name: "public");

            migrationBuilder.RenameTable(
                name: "Employees",
                newName: "Employees",
                newSchema: "public");

            migrationBuilder.RenameTable(
                name: "Departments",
                newName: "Departments",
                newSchema: "public");

            // 4. Меняем типы
            migrationBuilder.Sql(@"
                ALTER TABLE public.""Departments"" 
                ALTER COLUMN id DROP IDENTITY IF EXISTS;

                ALTER TABLE public.""Departments"" 
                ALTER COLUMN id TYPE text USING id::text;
            ");

            migrationBuilder.Sql(@"
                ALTER TABLE public.""Departments"" 
                ALTER COLUMN actual TYPE integer USING actual::integer;
            ");

            migrationBuilder.Sql(@"
                ALTER TABLE public.""Employees"" 
                ALTER COLUMN status_code TYPE integer USING status_code::integer;
            ");

            // 5. Устанавливаем NOT NULL и Default для tab_number
            migrationBuilder.AlterColumn<string>(
                name: "tab_number",
                schema: "public",
                table: "Employees",
                type: "text",
                nullable: false,
                defaultValue: "");

            // 6. Уникальный ключ по табельному номеру
            migrationBuilder.AddUniqueConstraint(
                name: "AK_Employees_tab_number",
                schema: "public",
                table: "Employees",
                column: "tab_number");

            // 7. Первичные ключи
            migrationBuilder.AddPrimaryKey(
                name: "employees_pkey",
                schema: "public",
                table: "Employees",
                column: "id");

            migrationBuilder.AddPrimaryKey(
                name: "departments2_pkey",
                schema: "public",
                table: "Departments",
                column: "id");

            migrationBuilder.RenameColumn(
                name: "parent_code",
                schema: "public",
                table: "Departments",
                newName: "parent_id");
            
            // 8. Внешние ключи
            migrationBuilder.AddForeignKey(
                name: "departments2_parent_id_fkey",
                schema: "public",
                table: "Departments",
                column: "parent_id",
                principalSchema: "public",
                principalTable: "Departments",
                principalColumn: "id",
                onDelete: ReferentialAction.SetNull);

            migrationBuilder.AddForeignKey(
                name: "employees_manager_tab_number_fkey",
                schema: "public",
                table: "Employees",
                column: "manager_tab_number",
                principalSchema: "public",
                principalTable: "Employees",
                principalColumn: "tab_number",
                onDelete: ReferentialAction.SetNull);
        }

        protected override void Down(MigrationBuilder migrationBuilder)
        {
            // Удаляем внешние ключи
            migrationBuilder.DropForeignKey(
                name: "departments2_parent_id_fkey",
                schema: "public",
                table: "Departments");

            migrationBuilder.DropForeignKey(
                name: "employees_manager_tab_number_fkey",
                schema: "public",
                table: "Employees");

            // Удаляем PK и UK
            migrationBuilder.DropUniqueConstraint(
                name: "AK_Employees_tab_number",
                schema: "public",
                table: "Employees");

            migrationBuilder.DropPrimaryKey(
                name: "employees_pkey",
                schema: "public",
                table: "Employees");

            migrationBuilder.DropPrimaryKey(
                name: "departments2_pkey",
                schema: "public",
                table: "Departments");

            // Возврат к старой схеме (если нужно)
            migrationBuilder.RenameTable(
                name: "Employees",
                schema: "public",
                newName: "Employees");

            migrationBuilder.RenameTable(
                name: "Departments",
                schema: "public",
                newName: "Departments");

            // Возврат к nullable и типам
            migrationBuilder.AlterColumn<string>(
                name: "tab_number",
                table: "Employees",
                type: "text",
                nullable: true);

            migrationBuilder.Sql(@"
                ALTER TABLE ""Employees"" 
                ALTER COLUMN status_code TYPE text USING status_code::text;
            ");

            migrationBuilder.Sql(@"
                ALTER TABLE ""Departments"" 
                ALTER COLUMN actual TYPE boolean USING actual::boolean;
            ");

            migrationBuilder.Sql(@"
                ALTER TABLE ""Departments"" 
                ALTER COLUMN id TYPE integer USING id::integer;
            ");

            // Восстановление первичных ключей (если нужно)
            migrationBuilder.AddPrimaryKey(
                name: "PK_Employees",
                table: "Employees",
                column: "id");

            migrationBuilder.AddPrimaryKey(
                name: "PK_departments",
                table: "Departments",
                column: "id");
        }
    }
}

