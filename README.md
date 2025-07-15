migrationBuilder.Sql(
    @"ALTER TABLE public.""Departments"" 
      ALTER COLUMN actual 
      TYPE integer 
      USING actual::integer;");
