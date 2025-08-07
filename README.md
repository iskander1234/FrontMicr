public class FileDto
    {
        /// <summary>Идентификатор (UUID) файла в системе</summary>
        public string FileId   { get; set; }

        /// <summary>Имя файла</summary>
        public string FileName { get; set; }

        /// <summary>Тип или расширение файла</summary>
        public string FileType { get; set; }
    }


    /// <summary>
    /// Событие создания записи в таблице ProcessFile
    /// </summary>
    public class ProcessFileCreatedEvent : BaseEntityEvent
    {
        /// <summary>Идентификатор связанного ProcessData</summary>
        public Guid ProcessDataId { get; set; }

        /// <summary>Имя файла</summary>
        public string FileName    { get; set; }

        /// <summary>Тип или расширение файла</summary>
        public string FileType    { get; set; }
    }
