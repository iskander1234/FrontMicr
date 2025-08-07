 Вот у меня есть сушествующий метод мне надо сюда интегрировать тот класс но как есть json и надо вотак расшрить "fileId" : "",
            "fileName" : ""
            "fileType" : ""
 а сам  выглядит так находится по запросу SELECT "PayloadJson" 
FROM public."ProcessData"
where "Id" ='ce361cd9-1691-44b3-a195-1a2a0fadb4e5'; "{""regData"":{""userCode"":""b.shymkentbay"",""userName"":""Шымкентбай Бақытжан Бахтиярұлы"",""departmentId"":""19.100512"",""departmentName"":""Управление разработки пенсионного учета"",""startdate"":""2025-08-05T10:09:46.584Z"",""regnum"":""""},""sysInfo"":{""userCode"":""b.shymkentbay"",""userName"":""Шымкентбай Бақытжан Бахтиярұлы"",""comment"":""comment"",""action"":""submit"",""condition"":""string""},""initiator"":{""id"":7820,""name"":""Шымкентбай Бақытжан Бахтиярұлы"",""position"":""Главный специалист"",""login"":""b.shymkentbay"",""statusCode"":6,""statusDescription"":""Работа"",""depId"":""19.100512"",""depName"":""Управление разработки пенсионного учета"",""parentDepId"":""19.100500"",""parentDepName"":""Департамент цифровизации"",""isFilial"":false,""mail"":""b.shymkentbay@enpf.kz"",""localPhone"":""0"",""mobilePhone"":""+7(708) 927-44-98"",""isManager"":false,""managerTabNumber"":""4340"",""disabled"":false,""tabNumber"":""00ЗП-00292""},""approvers"":[{""loginAD"":""m.ilespayev"",""id"":611,""name"":""Илеспаев Меииржан Анварович"",""shortName"":null,""position"":""Заместитель директора департамента"",""login"":""m.ilespayev"",""statusCode"":6,""statusDescription"":""Работа"",""depId"":""19.100500"",""depName"":""Департамент цифровизации"",""parentDepId"":""19.100500"",""parentDepName"":""Департамент цифровизации"",""isFilial"":false,""mail"":""m.ilespayev@enpf.kz"",""localPhone"":""0"",""mobilePhone"":""+7(702) 171-71-14"",""isManager"":true,""managerTabNumber"":""4303"",""disabled"":false,""tabNumber"":""00ЗП-00240""},{""loginAD"":""a.ysmail"",""id"":1545,""name"":""Ысмаил Арғынбек Байдабекұлы"",""shortName"":null,""position"":""Главный специалист"",""login"":""a.ysmail"",""statusCode"":6,""statusDescription"":""Работа"",""depId"":""19.100508"",""depName"":""Управление разработки фронтальных систем"",""parentDepId"":""19.100500"",""parentDepName"":""Департамент цифровизации"",""isFilial"":false,""mail"":""a.ysmail@enpf.kz"",""localPhone"":""0"",""mobilePhone"":""+7(702) 778-53-30"",""isManager"":false,""managerTabNumber"":""4340"",""disabled"":false,""tabNumber"":""00ЗП-00289""}],""recipients"":[{""loginAD"":""l.iskender"",""id"":633,""name"":""Искендер Лесхан Муратұлы"",""shortName"":null,""position"":""Главный специалист"",""login"":""l.iskender"",""statusCode"":6,""statusDescription"":""Работа"",""depId"":""19.100509"",""depName"":""Управление разработки Web приложений и сервисов"",""parentDepId"":""19.100500"",""parentDepName"":""Департамент цифровизации"",""isFilial"":false,""mail"":""l.iskender@enpf.kz"",""localPhone"":""0"",""mobilePhone"":""+7(707) 517-04-67"",""isManager"":false,""managerTabNumber"":""4340"",""disabled"":false,""tabNumber"":""00ЗП-00083""},{""loginAD"":""i.dosgali"",""id"":414,""name"":""Досгали Искандер Досгалиұлы"",""shortName"":null,""position"":""Главный специалист"",""login"":""i.dosgali"",""statusCode"":6,""statusDescription"":""Работа"",""depId"":""19.100509"",""depName"":""Управление разработки Web приложений и сервисов"",""parentDepId"":""19.100500"",""parentDepName"":""Департамент цифровизации"",""isFilial"":false,""mail"":""i.dosgali@enpf.kz"",""localPhone"":""0"",""mobilePhone"":""+7(747) 790-29-49"",""isManager"":false,""managerTabNumber"":""4340"",""disabled"":false,""tabNumber"":""00ЗП-00275""}],""signer"":{""loginAD"":""a.aristombekov"",""id"":168,""name"":""Аристомбеков Арстан Рамазанулы"",""shortName"":null,""position"":""Директор департамента"",""login"":""a.aristombekov"",""statusCode"":5,""statusDescription"":""Отпуск основной"",""depId"":""19.100500"",""depName"":""Департамент цифровизации"",""parentDepId"":""19.100500"",""parentDepName"":""Департамент цифровизации"",""isFilial"":false,""mail"":""a.aristombekov@enpf.kz"",""localPhone"":""0"",""mobilePhone"":""+7(705) 950-90-65"",""isManager"":true,""managerTabNumber"":""4303"",""disabled"":false,""tabNumber"":""4340""},""processData"":{""documentLang"":"""",""nomenclatureId"":""1"",""documentTitle"":""тема документа"",""grif"":""string"",""pageCount"":""5"",""signType"":""string"",""allGroup"":""string"",""meGroup"":""string"",""documentBody"":""<p><strong>ТЕКС&nbsp;СЗ</strong></p>""},""files"":null}"
  public class StartProcessCommandHandler(
    IUnitOfWork unitOfWork,
    IPayloadReaderService payloadReader,
    IProcessTaskService helperService,
    ICamundaService camundaService)
    : IRequestHandler<StartProcessCommand, BaseResponseDto<StartProcessResponse>>
    {

        public async Task<BaseResponseDto<StartProcessResponse>> Handle(StartProcessCommand command, CancellationToken cancellationToken)
        {
            var process = await unitOfWork.ProcessRepository.GetByFilterAsync(cancellationToken, p => p.ProcessCode == command.ProcessCode)
                ?? throw new HandlerException($"Процесс с кодом {command.ProcessCode} не найден", ErrorCodesEnum.Business);

            var requestNumber = await helperService.GenerateRequestNumberAsync(command.ProcessCode, cancellationToken);

            var options = new JsonSerializerOptions
            {
                Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping
            };

            var payloadJson = JsonSerializer.Serialize(command.Payload, options);
            var regData = payloadReader.ReadSection<RegData>(command.Payload, "regData");
            var processDataDto = payloadReader.ReadSection<ProcessDataDto>(command.Payload, "processData");

            var response = await camundaService.CamundaStartProcess(
               new CamundaStartProcessRequest
               {
                   processCode = command.ProcessCode
               });

            var processDataCreatedEvent = new ProcessDataCreatedEvent
            {
                ProcessId = process.Id,
                ProcessCode = process.ProcessCode,
                ProcessName = process.ProcessName,
                RegNumber = requestNumber,
                InitiatorCode = regData.UserCode,
                InitiatorName = regData.UserName,
                StatusCode = "Started",
                StatusName = "В работе",
                PayloadJson = payloadJson,
                Title = processDataDto.DocumentTitle,
                ProcessInstanceId = response
            };

            await unitOfWork.ProcessDataRepository.RaiseEvent(processDataCreatedEvent, cancellationToken);

            var processTaskHistoryCreatedEvent = new ProcessTaskHistoryCreatedEvent
            {
                ProcessDataId = processDataCreatedEvent.EntityId,
                TaskId = processDataCreatedEvent.EntityId, // Возможно стоит заменить на ID задачи, если будет доступен
                Action = ProcessAction.Start.ToString(),
                BlockName = "Регистрационная форма",
                Timestamp = DateTime.Now,
                PayloadJson = payloadJson,
                Comment = "",
                Description = "",
                ProcessCode = command.ProcessCode,
                ProcessName = process.ProcessName,
                RegNumber = requestNumber,
                InitiatorCode = regData.UserCode,
                InitiatorName = regData.UserName,
                Title = processDataDto.DocumentTitle
            };

            await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(processTaskHistoryCreatedEvent, cancellationToken);

            
            return new BaseResponseDto<StartProcessResponse>
            {
                Data = new StartProcessResponse
                {
                    ProcessGuid = processDataCreatedEvent.EntityId,
                    RegNumber = requestNumber
                }
            };
        }
