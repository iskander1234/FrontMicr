migrationBuilder.Sql(@"
    DO $$
    BEGIN
        IF NOT EXISTS (
            SELECT 1
            FROM pg_indexes
            WHERE indexname = 'IX_RefBusinessObjectAttributes_BusinessObjectId'
        ) THEN
            CREATE INDEX ""IX_RefBusinessObjectAttributes_BusinessObjectId""
            ON ""RefBusinessObjectAttributes""(""BusinessObjectId"");
        END IF;
    END$$;
");
