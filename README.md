info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (21ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      UPDATE public."Employees" SET tab_number = '' WHERE tab_number IS NULL;
      ALTER TABLE public."Employees" ALTER COLUMN tab_number SET NOT NULL;
      ALTER TABLE public."Employees" ALTER COLUMN tab_number SET DEFAULT '';
fail: Microsoft.EntityFrameworkCore.Database.Command[20102]
      Failed executing DbCommand (33ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      ALTER TABLE public."Employees" ALTER COLUMN status_code TYPE integer;
Failed executing DbCommand (33ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
ALTER TABLE public."Employees" ALTER COLUMN status_code TYPE integer;
Npgsql.PostgresException (0x80004005): 42804: столбец "status_code" нельзя автоматически привести к типу integer
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
    SqlState: 42804
    MessageText: столбец "status_code" нельзя автоматически привести к типу integer
    Hint: Возможно, необходимо указать "USING status_code::integer".
    File: tablecmds.c
    Line: 12941
    Routine: ATPrepAlterColumnType
42804: столбец "status_code" нельзя автоматически привести к типу integer



using Microsoft.EntityFrameworkCore.Migrations;
using Npgsql.EntityFrameworkCore.PostgreSQL.Metadata;

#nullable disable

namespace BpmBaseApi.Persistence.Migrations
{
    /// <inheritdoc />
    public partial class FixedTableEmployeesAndDepatments : Migration
    {
        /// <inheritdoc />
        protected override void Up(MigrationBuilder migrationBuilder)
        {
            migrationBuilder.DropPrimaryKey(
                name: "PK_Employees",
                table: "Employees");

            migrationBuilder.DropPrimaryKey(
                name: "PK_departments",
                table: "Departments");

            migrationBuilder.DropIndex(
                name: "IX_Departments_code",
                table: "Departments");

            migrationBuilder.DropColumn(
                name: "code",
                table: "Departments");

            migrationBuilder.EnsureSchema(
                name: "public");

            migrationBuilder.RenameTable(
                name: "Employees",
                newName: "Employees",
                newSchema: "public");

            migrationBuilder.RenameTable(
                name: "Departments",
                newName: "Departments",
                newSchema: "public");

            migrationBuilder.RenameColumn(
                name: "manager_id",
                schema: "public",
                table: "Employees",
                newName: "manager_tab_number");

            migrationBuilder.RenameColumn(
                name: "department_name",
                schema: "public",
                table: "Employees",
                newName: "parent_dep_name");

            migrationBuilder.RenameColumn(
                name: "department_id",
                schema: "public",
                table: "Employees",
                newName: "parent_dep_id");

            migrationBuilder.RenameIndex(
                name: "IX_Employees_TabNumber",
                schema: "public",
                table: "Employees",
                newName: "employees_tab_number_key");

            migrationBuilder.RenameColumn(
                name: "parent_code",
                schema: "public",
                table: "Departments",
                newName: "parent_id");

            migrationBuilder.AlterTable(
                name: "Employees",
                schema: "public",
                oldComment: "Справочник сотрудников");

            migrationBuilder.AlterTable(
                name: "Departments",
                schema: "public",
                oldComment: "Департаменты");

            migrationBuilder.AlterColumn<string>(
                name: "tab_number",
                schema: "public",
                table: "Employees",
                type: "text",
                nullable: false,
                defaultValue: "",
                oldClrType: typeof(string),
                oldType: "text",
                oldNullable: true);

            migrationBuilder.AlterColumn<int>(
                name: "status_code",
                schema: "public",
                table: "Employees",
                type: "integer",
                nullable: true,
                oldClrType: typeof(string),
                oldType: "text",
                oldNullable: true);

            migrationBuilder.AlterColumn<int>(
                name: "id",
                schema: "public",
                table: "Employees",
                type: "integer",
                nullable: false,
                defaultValueSql: "nextval('public.employees_id_seq'::regclass)",
                oldClrType: typeof(int),
                oldType: "integer",
                oldComment: "Первичный ключ")
                .OldAnnotation("Npgsql:ValueGenerationStrategy", NpgsqlValueGenerationStrategy.IdentityByDefaultColumn);

            migrationBuilder.AlterColumn<int>(
                name: "actual",
                schema: "public",
                table: "Departments",
                type: "integer",
                nullable: false,
                oldClrType: typeof(bool),
                oldType: "boolean");

            migrationBuilder.AlterColumn<string>(
                name: "id",
                schema: "public",
                table: "Departments",
                type: "text",
                nullable: false,
                oldClrType: typeof(int),
                oldType: "integer",
                oldComment: "Первичный ключ")
                .OldAnnotation("Npgsql:ValueGenerationStrategy", NpgsqlValueGenerationStrategy.IdentityByDefaultColumn);

            migrationBuilder.AddUniqueConstraint(
                name: "AK_Employees_tab_number",
                schema: "public",
                table: "Employees",
                column: "tab_number");

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

            migrationBuilder.CreateTable(
                name: "DepartmentCreatedEvent",
                columns: table => new
                {
                    Id = table.Column<string>(type: "text", nullable: false),
                    ParentId = table.Column<string>(type: "text", nullable: true),
                    Name = table.Column<string>(type: "text", nullable: false),
                    ManagerId = table.Column<string>(type: "text", nullable: true),
                    Actual = table.Column<int>(type: "integer", nullable: false),
                    DepartmentEntityId = table.Column<string>(type: "text", nullable: true)
                },
                constraints: table =>
                {
                    table.PrimaryKey("PK_DepartmentCreatedEvent", x => x.Id);
                    table.ForeignKey(
                        name: "FK_DepartmentCreatedEvent_Departments_DepartmentEntityId",
                        column: x => x.DepartmentEntityId,
                        principalSchema: "public",
                        principalTable: "Departments",
                        principalColumn: "id");
                });

            migrationBuilder.CreateIndex(
                name: "IX_Employees_manager_tab_number",
                schema: "public",
                table: "Employees",
                column: "manager_tab_number");

            migrationBuilder.CreateIndex(
                name: "IX_Departments_parent_id",
                schema: "public",
                table: "Departments",
                column: "parent_id");

            migrationBuilder.CreateIndex(
                name: "IX_DepartmentCreatedEvent_DepartmentEntityId",
                table: "DepartmentCreatedEvent",
                column: "DepartmentEntityId");

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

        /// <inheritdoc />
        protected override void Down(MigrationBuilder migrationBuilder)
        {
            migrationBuilder.DropForeignKey(
                name: "departments2_parent_id_fkey",
                schema: "public",
                table: "Departments");

            migrationBuilder.DropForeignKey(
                name: "employees_manager_tab_number_fkey",
                schema: "public",
                table: "Employees");

            migrationBuilder.DropTable(
                name: "DepartmentCreatedEvent");

            migrationBuilder.DropUniqueConstraint(
                name: "AK_Employees_tab_number",
                schema: "public",
                table: "Employees");

            migrationBuilder.DropPrimaryKey(
                name: "employees_pkey",
                schema: "public",
                table: "Employees");

            migrationBuilder.DropIndex(
                name: "IX_Employees_manager_tab_number",
                schema: "public",
                table: "Employees");

            migrationBuilder.DropPrimaryKey(
                name: "departments2_pkey",
                schema: "public",
                table: "Departments");

            migrationBuilder.DropIndex(
                name: "IX_Departments_parent_id",
                schema: "public",
                table: "Departments");

            migrationBuilder.RenameTable(
                name: "Employees",
                schema: "public",
                newName: "Employees");

            migrationBuilder.RenameTable(
                name: "Departments",
                schema: "public",
                newName: "Departments");

            migrationBuilder.RenameColumn(
                name: "parent_dep_name",
                table: "Employees",
                newName: "department_name");

            migrationBuilder.RenameColumn(
                name: "parent_dep_id",
                table: "Employees",
                newName: "department_id");

            migrationBuilder.RenameColumn(
                name: "manager_tab_number",
                table: "Employees",
                newName: "manager_id");

            migrationBuilder.RenameIndex(
                name: "employees_tab_number_key",
                table: "Employees",
                newName: "IX_Employees_TabNumber");

            migrationBuilder.RenameColumn(
                name: "parent_id",
                table: "Departments",
                newName: "parent_code");

            migrationBuilder.AlterTable(
                name: "Employees",
                comment: "Справочник сотрудников");

            migrationBuilder.AlterTable(
                name: "Departments",
                comment: "Департаменты");

            migrationBuilder.AlterColumn<string>(
                name: "tab_number",
                table: "Employees",
                type: "text",
                nullable: true,
                oldClrType: typeof(string),
                oldType: "text");

            migrationBuilder.AlterColumn<string>(
                name: "status_code",
                table: "Employees",
                type: "text",
                nullable: true,
                oldClrType: typeof(int),
                oldType: "integer",
                oldNullable: true);

            migrationBuilder.AlterColumn<int>(
                name: "id",
                table: "Employees",
                type: "integer",
                nullable: false,
                comment: "Первичный ключ",
                oldClrType: typeof(int),
                oldType: "integer",
                oldDefaultValueSql: "nextval('public.employees_id_seq'::regclass)")
                .Annotation("Npgsql:ValueGenerationStrategy", NpgsqlValueGenerationStrategy.IdentityByDefaultColumn);

            migrationBuilder.AlterColumn<bool>(
                name: "actual",
                table: "Departments",
                type: "boolean",
                nullable: false,
                oldClrType: typeof(int),
                oldType: "integer");

            migrationBuilder.AlterColumn<int>(
                name: "id",
                table: "Departments",
                type: "integer",
                nullable: false,
                comment: "Первичный ключ",
                oldClrType: typeof(string),
                oldType: "text")
                .Annotation("Npgsql:ValueGenerationStrategy", NpgsqlValueGenerationStrategy.IdentityByDefaultColumn);

            migrationBuilder.AddColumn<string>(
                name: "code",
                table: "Departments",
                type: "text",
                nullable: false,
                defaultValue: "");

            migrationBuilder.AddPrimaryKey(
                name: "PK_Employees",
                table: "Employees",
                column: "id");

            migrationBuilder.AddPrimaryKey(
                name: "PK_departments",
                table: "Departments",
                column: "id");

            migrationBuilder.CreateIndex(
                name: "IX_Departments_code",
                table: "Departments",
                column: "code",
                unique: true);
        }
    }
}

