dotnet ef migrations remove `
  --project BpmBaseApi.Persistence `
  --startup-project BpmBaseApi
  
dotnet ef database update 20250807050238_RemoveBlockKeyFromProcessTask `
  --project BpmBaseApi.Persistence `
  --startup-project BpmBaseApi


PS C:\BPM\bpm\bpmbaseapi> dotnet ef migrations remove `
>>   --project BpmBaseApi.Persistence `
>>   --startup-project BpmBaseApi
Build started...
Build succeeded.
warn: Microsoft.EntityFrameworkCore.Model.Validation[10400]
      Sensitive data logging is enabled. Log entries and exception messages may include sensitive application data; this mode should only be enabled during development.
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (77ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT "MigrationId", "ProductVersion"
      FROM "__EFMigrationsHistory"
      ORDER BY "MigrationId";
The migration '20250807113806_AddProcessFileTable' has already been applied to the database. Revert it and t
ry again. If the migration has been applied to other databases, consider reverting its changes using a new migration instead.
PS C:\BPM\bpm\bpmbaseapi> 



  // Persistence/Configurations/Entities/Process/ProcessFileConfiguration.cs
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using BpmBaseApi.Domain.Entities.Process;

namespace BpmBaseApi.Persistence.Configurations.Entities.Process
{
    public class ProcessFileConfiguration : IEntityTypeConfiguration<ProcessFileEntity>
    {
        public void Configure(EntityTypeBuilder<ProcessFileEntity> builder)
        {
            builder.ToTable("ProcessFile", t => t.HasComment("Файлы, прикреплённые к ProcessData"));

            // Указываем Id как PK
            builder.HasKey(p => p.Id);
            builder.Property(p => p.Id)
                   .HasComment("PK – GUID файла");

            builder.Property(p => p.FileName)
                   .IsRequired()
                   .HasMaxLength(255)
                   .HasComment("Имя файла");

            builder.Property(p => p.FileType)
                   .IsRequired()
                   .HasMaxLength(50)
                   .HasComment("Тип файла (расширение или MIME)");

            builder.Property(p => p.ProcessDataId)
                   .IsRequired()
                   .HasComment("FK → ProcessData.Id");

            builder.HasOne(p => p.ProcessData)
                   .WithMany(d => d.ProcessFiles)  // см. примечание ниже
                   .HasForeignKey(p => p.ProcessDataId)
                   .OnDelete(DeleteBehavior.Cascade);
        }
    }
}
