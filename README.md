migrationBuilder.Sql(@"
    ALTER TABLE public.""Departments""
    ALTER COLUMN id DROP IDENTITY IF EXISTS;

    ALTER TABLE public.""Departments""
    ALTER COLUMN id TYPE text USING id::text;
");
