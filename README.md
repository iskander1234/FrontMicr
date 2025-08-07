PS C:\BPM\bpm\bpmbaseapi> dotnet ef migrations add AddProcessFileTable --project BpmBaseApi.Persistence --startup-project BpmBaseApi
Build started...
Build succeeded.
warn: Microsoft.EntityFrameworkCore.Model.Validation[10400]
      Sensitive data logging is enabled. Log entries and exception messages may include sensitive application data; this mode should only be enabled during development.
Done. To undo this action, use 'ef migrations remove'
PS C:\BPM\bpm\bpmbaseapi> dotnet ef database update --project BpmBaseApi.Persistence --startup-project BpmBaseApi
Build started...
Build succeeded.
warn: Microsoft.EntityFrameworkCore.Model.Validation[10400]
      Sensitive data logging is enabled. Log entries and exception messages may include sensitive application data; this mode should only be enabled during development.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (60ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT "MigrationId", "ProductVersion"
      FROM "__EFMigrationsHistory"
      ORDER BY "MigrationId";
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (139ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      CREATE TABLE IF NOT EXISTS "__EFMigrationsHistory" (
          "MigrationId" character varying(150) NOT NULL,
          "ProductVersion" character varying(32) NOT NULL,
          CONSTRAINT "PK___EFMigrationsHistory" PRIMARY KEY ("MigrationId")
      );
info: Microsoft.EntityFrameworkCore.Migrations[20411]
      Acquiring an exclusive lock for migration application. See https://aka.ms/efcore-docs-migrations-lock for more information if this takes too long.
Acquiring an exclusive lock for migration application. See https://aka.ms/efcore-docs-migrations-lock for more information if this takes too long.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (39ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      LOCK TABLE "__EFMigrationsHistory" IN ACCESS EXCLUSIVE MODE
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (1ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT "MigrationId", "ProductVersion"
      FROM "__EFMigrationsHistory"
      ORDER BY "MigrationId";
info: Microsoft.EntityFrameworkCore.Migrations[20402]
      Applying migration '20250804103652_AddProcessInstanceId'.
Applying migration '20250804103652_AddProcessInstanceId'.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (11ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      ALTER TABLE "ProcessData" ADD "ProcessInstanceId" text;
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (9ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO "__EFMigrationsHistory" ("MigrationId", "ProductVersion")
      VALUES ('20250804103652_AddProcessInstanceId', '9.0.4');
info: Microsoft.EntityFrameworkCore.Migrations[20402]
      Applying migration '20250805110215_AddRefProcessStage'.
Applying migration '20250805110215_AddRefProcessStage'.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (158ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      CREATE TABLE "RefProcessStage" (
          "Id" uuid NOT NULL,
          "Code" character varying(200) NOT NULL,
          "Name" character varying(100) NOT NULL,
          "Created" timestamp without time zone NOT NULL,
          "Updated" timestamp without time zone NOT NULL,
          "CreateUserId" text NOT NULL,
          "LastUserId" text NOT NULL,
          CONSTRAINT "PK_RefProcessStage" PRIMARY KEY ("Id")
      );
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (1ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO "__EFMigrationsHistory" ("MigrationId", "ProductVersion")
      VALUES ('20250805110215_AddRefProcessStage', '9.0.4');
info: Microsoft.EntityFrameworkCore.Migrations[20402]
      Applying migration '20250807050238_RemoveBlockKeyFromProcessTask'.
Applying migration '20250807050238_RemoveBlockKeyFromProcessTask'.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (29ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      ALTER TABLE "ProcessTask" DROP CONSTRAINT "FK_ProcessTask_Block_BlockId";
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (20ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      DROP INDEX "IX_ProcessTask_BlockId";
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (1ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      ALTER TABLE "ProcessTask" ALTER COLUMN "BlockId" DROP NOT NULL;
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (1ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO "__EFMigrationsHistory" ("MigrationId", "ProductVersion")
      VALUES ('20250807050238_RemoveBlockKeyFromProcessTask', '9.0.4');
info: Microsoft.EntityFrameworkCore.Migrations[20402]
      Applying migration '20250807113806_AddProcessFileTable'.
Applying migration '20250807113806_AddProcessFileTable'.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (66ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      CREATE TABLE "ProcessFile" (
          "FileId" uuid NOT NULL,
          "FileName" character varying(255) NOT NULL,
          "FileType" character varying(50) NOT NULL,
          "ProcessDataId" uuid NOT NULL,
          "Id" uuid NOT NULL,
          "Created" timestamp without time zone NOT NULL,
          "Updated" timestamp without time zone NOT NULL,
          "CreateUserId" text NOT NULL,
          "LastUserId" text NOT NULL,
          CONSTRAINT "PK_ProcessFile" PRIMARY KEY ("FileId"),
          CONSTRAINT "FK_ProcessFile_ProcessData_ProcessDataId" FOREIGN KEY ("ProcessDataId") REFERENCES "ProcessData" ("Id") ON DELETE CASCADE
      );
      COMMENT ON TABLE "ProcessFile" IS 'Файлы, прикреплённые к ProcessData';
      COMMENT ON COLUMN "ProcessFile"."FileId" IS 'PK - GUID файла';
      COMMENT ON COLUMN "ProcessFile"."FileName" IS 'Имя файла';
      COMMENT ON COLUMN "ProcessFile"."FileType" IS 'Тип файла (расширение или MIME)';
      COMMENT ON COLUMN "ProcessFile"."ProcessDataId" IS 'FK ⸮ ProcessData.Id';
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (14ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      CREATE INDEX "IX_ProcessFile_ProcessDataId" ON "ProcessFile" ("ProcessDataId");
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (1ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO "__EFMigrationsHistory" ("MigrationId", "ProductVersion")
      VALUES ('20250807113806_AddProcessFileTable', '9.0.4');
Done.
PS C:\BPM\bpm\bpmbaseapi> 
