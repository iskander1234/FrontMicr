// Добавляем временное поле
migrationBuilder.AddColumn<int>(
    name: "status_code_int",
    schema: "public",
    table: "Employees",
    type: "integer",
    nullable: true);

// Обновляем значения (если только числа)
migrationBuilder.Sql(@"
    UPDATE public.""Employees""
    SET status_code_int = NULLIF(status_code, '')::integer
    WHERE status_code ~ '^\d+$';
");

// Удаляем старый текстовый столбец
migrationBuilder.DropColumn(
    name: "status_code",
    schema: "public",
    table: "Employees");

// Переименовываем новый в оригинальное имя
migrationBuilder.RenameColumn(
    name: "status_code_int",
    schema: "public",
    table: "Employees",
    newName: "status_code");
