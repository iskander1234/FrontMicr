migrationBuilder.Sql(@"
    -- Удалить PRIMARY KEY (временно)
    ALTER TABLE public.""Departments"" DROP CONSTRAINT IF EXISTS departments2_pkey;

    -- Удалить IDENTITY
    ALTER TABLE public.""Departments"" ALTER COLUMN id DROP IDENTITY IF EXISTS;

    -- Изменить тип id
    ALTER TABLE public.""Departments"" ALTER COLUMN id TYPE text USING id::text;

    -- Вернуть PRIMARY KEY
    ALTER TABLE public.""Departments"" ADD CONSTRAINT departments2_pkey PRIMARY KEY (id);
");
