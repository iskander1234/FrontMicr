// Domain/Entities/Process/ProcessFileEntity.cs
using System;
using BpmBaseApi.Domain.SeedWork;

namespace BpmBaseApi.Domain.Entities.Process
{
    /// <summary>
    /// Файлы, прикреплённые к заявке (ProcessData)
    /// </summary>
    public class ProcessFileEntity : BaseJournaledEntity
    {
        /// <summary>PK – GUID файла</summary>
        public Guid FileId { get; set; }

        /// <summary>Имя файла</summary>
        public string FileName { get; set; }

        /// <summary>Тип файла (расширение или MIME)</summary>
        public string FileType { get; set; }

        /// <summary>FK → ProcessData.Id</summary>
        public Guid ProcessDataId { get; set; }

        // Навигация (необязательно, но полезно)
        public ProcessDataEntity ProcessData { get; set; }
    }
}



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

            // PK
            builder.HasKey(p => p.FileId);
            builder.Property(p => p.FileId)
                   .HasComment("PK – GUID файла");

            // FileName
            builder.Property(p => p.FileName)
                   .IsRequired()
                   .HasMaxLength(255)
                   .HasComment("Имя файла");

            // FileType
            builder.Property(p => p.FileType)
                   .IsRequired()
                   .HasMaxLength(50)
                   .HasComment("Тип файла (расширение или MIME)");

            // ProcessDataId + FK
            builder.Property(p => p.ProcessDataId)
                   .IsRequired()
                   .HasComment("FK → ProcessData.Id");

            builder.HasOne(p => p.ProcessData)
                   .WithMany(d => d.ProcessTasks) // или .WithMany(d => d.Files), если вы добавите в ProcessDataEntity свойство List<ProcessFileEntity> Files
                   .HasForeignKey(p => p.ProcessDataId)
                   .OnDelete(DeleteBehavior.Cascade);

            // Если в ProcessDataEntity нет коллекции Files, то можно просто не настраивать навигацию
        }
    }
}
