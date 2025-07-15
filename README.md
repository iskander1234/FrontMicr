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
