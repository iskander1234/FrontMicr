Нужно создать новую таблицу 

Таблица ProcessFile
fileId- uuid
FileName - string
FileType - string 
ProcessDataId – Ид таблицы брать надо из таблицы  ProcessData  - uuid 




using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.SeedWork;

namespace BpmBaseApi.Domain.Entities.Process
{
    public class ProcessDataEntity : BaseJournaledEntity
    {
        public Guid Id { get; set; }
        public Guid ProcessId { get; set; }
        public ProcessEntity? Process { get; set; }
        public string ProcessCode { get; set; }
        public string ProcessName { get; set; }
        public string RegNumber { get; set; } 
        public string StatusCode { get; set; }
        public string StatusName { get; set; }
        public string InitiatorCode { get; set; }
        public string InitiatorName { get; set; }
        public string PayloadJson { get; set; }
        public string? Title { get; set; }
        public string? BlockCode { get; set; }
        public string? BlockName { get; set; }
        public string? ProcessInstanceId { get; set; }
        public List<ProcessTaskEntity> ProcessTasks { get; set; } = new();
        public List<ProcessTaskHistoryEntity> History { get; set; } = new();

        #region Apply

        public void Apply(ProcessDataCreatedEvent @event)
        {
            Id = Guid.NewGuid();
            @event.EntityId = Id;
            ProcessId = @event.ProcessId;
            ProcessCode = @event.ProcessCode;
            ProcessName = @event.ProcessName;
            RegNumber = @event.RegNumber;
            StatusCode = @event.StatusCode;
            StatusName = @event.StatusName;
            InitiatorCode = @event.InitiatorCode;
            InitiatorName = @event.InitiatorName;
            PayloadJson = @event.PayloadJson;
            Title = @event.Title; 
            ProcessInstanceId = @event.ProcessInstanceId;
        }

        public void Apply(ProcessDataStatusChangedEvent @event)
        {
            StatusCode = @event.StatusCode;
            StatusName = @event.StatusName;
        }

        public void Apply(ProcessDataBlockChangedEvent @event)
        {
            BlockCode = @event.BlockCode;
            BlockName = @event.BlockName;
        }

        #endregion
    }
}

using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Microsoft.EntityFrameworkCore;
using BpmBaseApi.Domain.Entities.Process;

namespace BpmBaseApi.Persistence.Configurations.Entities.Process
{
    public class ProcessDataConfiguration : IEntityTypeConfiguration<ProcessDataEntity>
    {
        public void Configure(EntityTypeBuilder<ProcessDataEntity> builder)
        {
            builder.ToTable("ProcessData", t => t.HasComment("Заявки, проходящие через процессы"));

            builder
              .Property(p => p.ProcessId)
              .HasComment("Идентификатор заявки");
            builder
                .HasOne(p => p.Process)
                .WithMany()
                .HasForeignKey(p => p.ProcessId)
                .OnDelete(DeleteBehavior.Restrict);
            builder
                .Navigation(p => p.Process);

            builder.Property(d => d.RegNumber)
                .HasMaxLength(50)
                .HasComment("Регистрационный номер заявки");

            builder.Property(d => d.StatusCode)
                .HasMaxLength(50)
                .HasComment("Статус заявки");

            builder.Property(d => d.PayloadJson)
                .HasComment("Общие данные формы заявки в JSON");

            //builder.Navigation(d => d.Process).AutoInclude(false);
            //builder.Navigation(d => d.ProcessTasks).AutoInclude(false);
        }
    }

}



{"regData":{"userCode":"b.shymkentbay","userName":"Шымкентбай Бақытжан Бахтиярұлы","departmentId":"19.100512","departmentName":"Управление разработки пенсионного учета","startdate":"2025-08-05T10:09:46.584Z","regnum":""},"sysInfo":{"userCode":"b.shymkentbay","userName":"Шымкентбай Бақытжан Бахтиярұлы","comment":"comment","action":"submit","condition":"string"},"initiator":{"id":7820,"name":"Шымкентбай Бақытжан Бахтиярұлы","position":"Главный специалист","login":"b.shymkentbay","statusCode":6,"statusDescription":"Работа","depId":"19.100512","depName":"Управление разработки пенсионного учета","parentDepId":"19.100500","parentDepName":"Департамент цифровизации","isFilial":false,"mail":"b.shymkentbay@enpf.kz","localPhone":"0","mobilePhone":"+7(708) 927-44-98","isManager":false,"managerTabNumber":"4340","disabled":false,"tabNumber":"00ЗП-00292"},"approvers":[{"loginAD":"m.ilespayev","id":611,"name":"Илеспаев Меииржан Анварович","shortName":null,"position":"Заместитель директора департамента","login":"m.ilespayev","statusCode":6,"statusDescription":"Работа","depId":"19.100500","depName":"Департамент цифровизации","parentDepId":"19.100500","parentDepName":"Департамент цифровизации","isFilial":false,"mail":"m.ilespayev@enpf.kz","localPhone":"0","mobilePhone":"+7(702) 171-71-14","isManager":true,"managerTabNumber":"4303","disabled":false,"tabNumber":"00ЗП-00240"},{"loginAD":"a.ysmail","id":1545,"name":"Ысмаил Арғынбек Байдабекұлы","shortName":null,"position":"Главный специалист","login":"a.ysmail","statusCode":6,"statusDescription":"Работа","depId":"19.100508","depName":"Управление разработки фронтальных систем","parentDepId":"19.100500","parentDepName":"Департамент цифровизации","isFilial":false,"mail":"a.ysmail@enpf.kz","localPhone":"0","mobilePhone":"+7(702) 778-53-30","isManager":false,"managerTabNumber":"4340","disabled":false,"tabNumber":"00ЗП-00289"}],"recipients":[{"loginAD":"l.iskender","id":633,"name":"Искендер Лесхан Муратұлы","shortName":null,"position":"Главный специалист","login":"l.iskender","statusCode":6,"statusDescription":"Работа","depId":"19.100509","depName":"Управление разработки Web приложений и сервисов","parentDepId":"19.100500","parentDepName":"Департамент цифровизации","isFilial":false,"mail":"l.iskender@enpf.kz","localPhone":"0","mobilePhone":"+7(707) 517-04-67","isManager":false,"managerTabNumber":"4340","disabled":false,"tabNumber":"00ЗП-00083"},{"loginAD":"i.dosgali","id":414,"name":"Досгали Искандер Досгалиұлы","shortName":null,"position":"Главный специалист","login":"i.dosgali","statusCode":6,"statusDescription":"Работа","depId":"19.100509","depName":"Управление разработки Web приложений и сервисов","parentDepId":"19.100500","parentDepName":"Департамент цифровизации","isFilial":false,"mail":"i.dosgali@enpf.kz","localPhone":"0","mobilePhone":"+7(747) 790-29-49","isManager":false,"managerTabNumber":"4340","disabled":false,"tabNumber":"00ЗП-00275"}],"signer":{"loginAD":"a.aristombekov","id":168,"name":"Аристомбеков Арстан Рамазанулы","shortName":null,"position":"Директор департамента","login":"a.aristombekov","statusCode":5,"statusDescription":"Отпуск основной","depId":"19.100500","depName":"Департамент цифровизации","parentDepId":"19.100500","parentDepName":"Департамент цифровизации","isFilial":false,"mail":"a.aristombekov@enpf.kz","localPhone":"0","mobilePhone":"+7(705) 950-90-65","isManager":true,"managerTabNumber":"4303","disabled":false,"tabNumber":"4340"},"processData":{"documentLang":"","nomenclatureId":"1","documentTitle":"тема документа","grif":"string","pageCount":"5","signType":"string","allGroup":"string","meGroup":"string","documentBody":"<p><strong>ТЕКС&nbsp;СЗ</strong></p>"},"files":null}
