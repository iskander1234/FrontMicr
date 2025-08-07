dotnet ef migrations remove `
  --project BpmBaseApi.Persistence `
  --startup-project BpmBaseApi


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
