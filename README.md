migrationBuilder.Sql(@"
    ALTER TABLE public.""Employees""
    ALTER COLUMN status_code TYPE integer USING status_code::integer;
");
