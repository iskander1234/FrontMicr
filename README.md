migrationBuilder.Sql(@"
    -- Удалить PRIMARY KEY (временно)
    ALTER TABLE public.""Departments"" DROP CONSTRAINT IF EXISTS departments2_pkey;

    -- Удалить sequence / identity привязку
    ALTER TABLE public.""Departments"" ALTER COLUMN id DROP DEFAULT;
    DROP SEQUENCE IF EXISTS public.departments_id_seq;

    -- Изменить тип поля id
    ALTER TABLE public.""Departments"" ALTER COLUMN id TYPE text USING id::text;

    -- Вернуть PRIMARY KEY
    ALTER TABLE public.""Departments"" ADD CONSTRAINT departments2_pkey PRIMARY KEY (id);
");
