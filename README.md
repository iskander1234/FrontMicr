migrationBuilder.Sql(@"
    UPDATE public.""Departments""
    SET parent_id = NULL
    WHERE parent_id IS NOT NULL
      AND parent_id NOT IN (
          SELECT id FROM public.""Departments""
      );
");
