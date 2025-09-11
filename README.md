Вот есть база SELECT * FROM public."ProcessData"
where "Id" ='9f68363a-1a86-46f2-ad1e-8a6fa23de72a' 

 таблица у нас ProcessDataEntity нужно создать хендлер и проверять вернуть true and false из json она находиться тут using BpmBaseApi.Domain.Entities.Event.Process;
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

        public void Apply(ProcessDataProcessInstanseIdChangedEvent @event)
        {
            ProcessInstanceId = @event.ProcessInstanceId;
        }
        
        public void Apply(ProcessDataEditedEvent @event)
        {
            StatusCode    = @event.StatusCode!;
            StatusName    = @event.StatusName!;
            PayloadJson   = @event.PayloadJson!; 
            InitiatorCode = @event.InitiatorCode!; 
            InitiatorName = @event.InitiatorName!;
            Title         = @event.Title!;
        }

        #endregion
    }
}
 именно в PayloadJson запись такая смотрим исенно сюда  "translatable": false, если false то false а если тут true то true {
  "regData": {
    "userCode": "m.ilespayev",
    "userName": "Илеспаев Меииржан Анварович",
    "departmentId": "00ЗП-0013",
    "departmentName": "Управление автоматизации бизнес-процессов",
    "startdate": "2025-09-11T12:12:20.674Z",
    "regnum": "OR-0020-2025"
  },
  "sysInfo": {
    "userCode": "m.ilespayev",
    "userName": "Илеспаев Меииржан Анварович",
    "comment": "comment",
    "action": "submit",
    "condition": "string"
  },
  "initiator": {
    "id": 611,
    "name": "Илеспаев Меииржан Анварович",
    "shortName": "Илеспаев М.А.",
    "position": "Начальник управления",
    "login": "m.ilespayev",
    "statusCode": 6,
    "statusDescription": "Работа",
    "depId": "00ЗП-0013",
    "depName": "Управление автоматизации бизнес-процессов",
    "parentDepId": "00ЗП-0010",
    "parentDepName": "Департамент цифровой трансформации",
    "isFilial": false,
    "mail": "m.ilespayev@enpf.kz",
    "localPhone": "0234",
    "mobilePhone": "+7(702) 171-71-14",
    "isManager": true,
    "managerTabNumber": "4340",
    "disabled": false,
    "tabNumber": "00ЗП-00240",
    "loginAD": "m.ilespayev"
  },
  "approvers": [
    {
      "loginAD": "m.ilespayev",
      "id": 611,
      "name": "Илеспаев Меииржан Анварович",
      "shortName": "Илеспаев М.А.",
      "position": "Начальник управления",
      "login": "m.ilespayev",
      "statusCode": 6,
      "statusDescription": "Работа",
      "depId": "00ЗП-0013",
      "depName": "Управление автоматизации бизнес-процессов",
      "parentDepId": "00ЗП-0010",
      "parentDepName": "Департамент цифровой трансформации",
      "isFilial": false,
      "mail": "m.ilespayev@enpf.kz",
      "localPhone": "0",
      "mobilePhone": "+7(702) 171-71-14",
      "isManager": true,
      "managerTabNumber": "4340",
      "disabled": false,
      "tabNumber": "00ЗП-00240",
      "order": null
    }
  ],
  "recipients": [
    {
      "loginAD": "m.ilespayev",
      "id": 611,
      "name": "Илеспаев Меииржан Анварович",
      "shortName": "Илеспаев М.А.",
      "position": "Начальник управления",
      "login": "m.ilespayev",
      "statusCode": 6,
      "statusDescription": "Работа",
      "depId": "00ЗП-0013",
      "depName": "Управление автоматизации бизнес-процессов",
      "parentDepId": "00ЗП-0010",
      "parentDepName": "Департамент цифровой трансформации",
      "isFilial": false,
      "mail": "m.ilespayev@enpf.kz",
      "localPhone": "0",
      "mobilePhone": "+7(702) 171-71-14",
      "isManager": true,
      "managerTabNumber": "4340",
      "disabled": false,
      "tabNumber": "00ЗП-00240"
    },
    {
      "loginAD": "a.ysmail",
      "id": 1545,
      "name": "Ысмаил Арғынбек Байдабекұлы",
      "shortName": "Ысмаил А.Б.",
      "position": "Главный специалист",
      "login": "a.ysmail",
      "statusCode": 6,
      "statusDescription": "Работа",
      "depId": "00ЗП-0013",
      "depName": "Управление автоматизации бизнес-процессов",
      "parentDepId": "00ЗП-0010",
      "parentDepName": "Департамент цифровой трансформации",
      "isFilial": false,
      "mail": "a.ysmail@enpf.kz",
      "localPhone": "0",
      "mobilePhone": "+7(702) 778-53-30",
      "isManager": false,
      "managerTabNumber": "4340",
      "disabled": false,
      "tabNumber": "00ЗП-00289"
    }
  ],
  "signer": {
    "loginAD": "m.ilespayev",
    "id": 611,
    "name": "Илеспаев Меииржан Анварович",
    "shortName": "Илеспаев М.А.",
    "position": "Начальник управления",
    "login": "m.ilespayev",
    "statusCode": 6,
    "statusDescription": "Работа",
    "depId": "00ЗП-0013",
    "depName": "Управление автоматизации бизнес-процессов",
    "parentDepId": "00ЗП-0010",
    "parentDepName": "Департамент цифровой трансформации",
    "isFilial": false,
    "mail": "m.ilespayev@enpf.kz",
    "localPhone": "0",
    "mobilePhone": "+7(702) 171-71-14",
    "isManager": true,
    "managerTabNumber": "4340",
    "disabled": false,
    "tabNumber": "00ЗП-00240"
  },
  "processData": {
    "confLevelCode": "21ecd97e-779d-4efe-b522-8cc1c6fa4b9f",
    "confLevelName": "string",
    "documentTypeCode": "order",
    "documentTypeName": "Приказ",
    "documentTitle": "Тема приказа",
    "dueDate": "Tue Sep 16 2025",
    "documentLang": "ru",
    "translatable": false,
    "allGroups": "",
    "myGroups": "",
    "documentBody": "<p class=\"ql-align-center\">\t<strong>ПРОТОКОЛ&nbsp;№&nbsp;3</strong></p><p class=\"ql-align-center\"><strong>&nbsp;очного&nbsp;заседания&nbsp;Риск-комитета&nbsp;АО&nbsp;«ЕНПФ»</strong></p><p class=\"ql-align-justify\">\t</p><p class=\"ql-align-justify\">\t<strong>г.&nbsp;Алматы&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;«23»&nbsp;июля&nbsp;2025&nbsp;года</strong></p><p class=\"ql-align-justify\">\t</p><p class=\"ql-align-justify\">\t<strong>Присутствовали:</strong></p><p class=\"ql-align-justify\">\tПредседатель&nbsp;Риск-комитета&nbsp;–&nbsp;Егеубаева&nbsp;С.А.&nbsp;(Заместитель&nbsp;Председателя&nbsp;Правления)</p><p class=\"ql-align-justify\">\tЧлены&nbsp;Риск-комитета:</p><p class=\"ql-align-justify\">\tТулегенова&nbsp;Ж.К.,&nbsp;Управляющий&nbsp;директор;</p><p class=\"ql-align-justify\">\tБозжанов&nbsp;Н.С.,&nbsp;Управляющий&nbsp;директор;</p><p class=\"ql-align-justify\">\tСадырбаев&nbsp;Р.К.,&nbsp;и.о.&nbsp;директора&nbsp;Департамента&nbsp;безопасности&nbsp;(далее&nbsp;–&nbsp;ДБ);</p><p class=\"ql-align-justify\">\tБактыбаев&nbsp;Н.А.,&nbsp;директор&nbsp;Юридического&nbsp;департамента&nbsp;(далее&nbsp;–&nbsp;ЮД);</p><p class=\"ql-align-justify\">\tФазылова&nbsp;К.Н.,&nbsp;директор&nbsp;Департамента&nbsp;стратегического&nbsp;развития&nbsp;(далее&nbsp;–&nbsp;ДСР);</p><p class=\"ql-align-justify\">\tОспан&nbsp;Э.Қ.,&nbsp;директор&nbsp;Департамента&nbsp;риск-менеджмента&nbsp;(далее&nbsp;–&nbsp;ДРМ);</p><p>\tБайкова&nbsp;Г.К.,&nbsp;и.о.&nbsp;директора&nbsp;Департамента&nbsp;кибербезопасности&nbsp;(далее&nbsp;–&nbsp;ДКБ).</p><p class=\"ql-align-justify\">\t</p><p class=\"ql-align-justify\">\tПриглашенные&nbsp;лица:</p><p class=\"ql-align-justify\">\tАуталипова&nbsp;М.С.,&nbsp;главный&nbsp;специалист&nbsp;Управления&nbsp;проектов&nbsp;Департамента&nbsp;цифровизации&nbsp;(далее&nbsp;–&nbsp;ДЦ);</p><p class=\"ql-align-justify\">\tСадвакасов&nbsp;Т.Т.,&nbsp;директор&nbsp;Департамента&nbsp;развития&nbsp;и&nbsp;поддержки&nbsp;инфраструктуры&nbsp;(далее&nbsp;–&nbsp;ДРПИ).</p><p class=\"ql-align-justify\">\t</p><p class=\"ql-align-justify\">\tСекретарь&nbsp;Риск-комитета:</p><p class=\"ql-align-justify\">\tТашанова&nbsp;Светлана&nbsp;Маратовна&nbsp;–&nbsp;главный&nbsp;специалист&nbsp;Отдела&nbsp;операционных&nbsp;рисков&nbsp;ДРМ.</p><p class=\"ql-align-justify\">\tСекретарь&nbsp;Риск-комитета&nbsp;сообщил&nbsp;о&nbsp;наличии&nbsp;кворума&nbsp;(100%)&nbsp;для&nbsp;проведения&nbsp;заседания&nbsp;Риск-комитета&nbsp;АО&nbsp;«ЕНПФ»&nbsp;(далее&nbsp;–&nbsp;Фонд).</p><p class=\"ql-align-justify\">\t</p><p class=\"ql-align-justify\">\t<strong>Повестка&nbsp;дня:</strong></p><p class=\"ql-align-justify\">\t1)&nbsp;&nbsp;&nbsp;&nbsp;Рассмотрение&nbsp;и&nbsp;одобрение&nbsp;Аналитической&nbsp;записки&nbsp;по&nbsp;операционным&nbsp;рискам&nbsp;за&nbsp;2&nbsp;квартал&nbsp;2025&nbsp;года&nbsp;(докладчик&nbsp;директор&nbsp;ДРМ&nbsp;Оспан&nbsp;Э.Қ.);</p><p class=\"ql-align-justify\">\t2)&nbsp;&nbsp;&nbsp;&nbsp;Рассмотрение&nbsp;и&nbsp;одобрение&nbsp;Информации&nbsp;о&nbsp;соблюдении&nbsp;(использовании)&nbsp;требований&nbsp;системы&nbsp;управления&nbsp;рисками&nbsp;АО&nbsp;«ЕНПФ»,&nbsp;о&nbsp;карте&nbsp;рисков,&nbsp;анализе&nbsp;влияния&nbsp;ключевых&nbsp;рисков&nbsp;на&nbsp;достижение&nbsp;стратегических&nbsp;целей&nbsp;Фонда,&nbsp;исполнении&nbsp;мероприятий&nbsp;по&nbsp;реагированию&nbsp;на&nbsp;риски&nbsp;(минимизации&nbsp;рисков)&nbsp;и&nbsp;результатах&nbsp;мониторинга&nbsp;соблюдения&nbsp;уровня&nbsp;риск-аппетита&nbsp;и&nbsp;риск-толерантности&nbsp;по&nbsp;собственным&nbsp;активам&nbsp;АО&nbsp;«ЕНПФ»&nbsp;за&nbsp;2&nbsp;квартал&nbsp;2025&nbsp;года&nbsp;(докладчик&nbsp;директор&nbsp;ДРМ&nbsp;Оспан&nbsp;Э.Қ.);</p><p class=\"ql-align-justify\">\t3)&nbsp;Рассмотрение&nbsp;и&nbsp;одобрение&nbsp;Отчетов&nbsp;о&nbsp;состоянии&nbsp;ИТ-инфраструктуры&nbsp;и&nbsp;развитии&nbsp;информационных&nbsp;систем&nbsp;и&nbsp;электронных&nbsp;услуг&nbsp;АО&nbsp;&quot;ЕНПФ&quot;&nbsp;за&nbsp;2&nbsp;квартал&nbsp;2025&nbsp;года&nbsp;(докладчики:&nbsp;директор&nbsp;ДРПИ&nbsp;Садвакасов&nbsp;Т.Т.,&nbsp;главный&nbsp;специалист&nbsp;Управления&nbsp;проектов&nbsp;ДЦ&nbsp;Ауталипова&nbsp;М.С.);</p><p class=\"ql-align-justify\">\t4)&nbsp;Рассмотрение&nbsp;и&nbsp;одобрение&nbsp;сведений&nbsp;(информации)&nbsp;по&nbsp;выявленным&nbsp;инцидентам&nbsp;(нарушениям)&nbsp;информационной&nbsp;безопасности&nbsp;(докладчики:&nbsp;и.о.&nbsp;директора&nbsp;ДКБ&nbsp;Байкова&nbsp;Г.К.).</p><p class=\"ql-align-justify\">\t</p><p class=\"ql-align-justify\">\tПредседателем&nbsp;Риск-комитета&nbsp;было&nbsp;предложено&nbsp;проголосовать&nbsp;за&nbsp;утверждение&nbsp;повестки&nbsp;дня.&nbsp;</p><p></p>"
  },
  "files": [
    {
      "fileId": "8dc7ec34-ef88-4fb0-b4c9-3cda09c6012d",
      "fileName": "тест вложения.docx",
      "fileType": "Attachment"
    },
    {
      "fileId": "6da67032-2abf-41b3-8ba0-5f90b5653482",
      "fileName": "Номенклатура ДЦТ.docx",
      "fileType": "Attachment"
    }
  ]
}
    
    /// <summary>
        ///     Проверяет нужна ли перевод
        /// </summary>
        /// <param name="command">Индивидуальный идентификатор процесса</param>
        /// <param name="cancellationToken">Маркер отмены, используемый для отмены HTTP-запроса</param>
        /// <returns></returns>
        [HttpPost("needsTranslations")]
        [SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<NeedsTranslationResponse>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        [SwaggerResponse(StatusCodes.Status503ServiceUnavailable)]
        public async Task<IActionResult> CheckNeedTranslations([FromBody] NeedsTranslationCommand command, CancellationToken cancellationToken)
        {
            //Не до конца реализован
            //Надо создать обработчик и результат (класс)
            return Ok(new BaseResponseDto<NeedsTranslationResponse> { Data = new NeedsTranslationResponse() { Result = true } });
        }




        using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Shared.Commands.Process
{
    public class NeedsTranslationCommand : IRequest<BaseResponseDto<NeedsTranslationResponse>>
    {
        public Guid ProcessGuid { get; set; }
    }
}



