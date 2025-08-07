 // EF + Журнал требуют публичный конструктор без параметров
        public ProcessFileEntity() { }

        #region Apply

        /// <summary>Заполнение полей при ProcessFileCreatedEvent</summary>
        public void Apply(ProcessFileCreatedEvent @event)
        {
            // @event.EntityId берётся из RaiseEvent(...).EntityId
            Id            = @event.EntityId;
            FileName      = @event.FileName;
            FileType      = @event.FileType;
            ProcessDataId = @event.ProcessDataId;
        }

        /// <summary>При удалении файла ничего не нужно делать —
        /// репозиторий вызовет DeleteByIdAsync автоматически</summary>
        public void Apply(ProcessFileDeletedEvent @event)
        {
            // пусто
        }

        #endregion
    }
