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
