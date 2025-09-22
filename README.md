using System.Text.Encodings.Web;
using System.Text.Json;
using System.Text.Json.Nodes;
using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Responses.Process;
using BpmBaseApi.Shared.Models.Camunda;
using BpmBaseApi.Shared.Dtos;
using MediatR;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    /// <summary>
    /// Упрощённый старт под ITSM:
    /// - без поиска/переиспользования Draft/Started
    /// - не сливаем payload, не редактируем существующие записи
    /// - создаём ProcessData=Started, стартуем Camunda, апсертим files из payload, пишем Start в историю
    /// </summary>
    public class StartProcessITSMCommandHandler(
        IUnitOfWork unitOfWork,
        IProcessTaskService helperService,
        ICamundaService camundaService
    ) : IRequestHandler<StartProcessITSMCommand, BaseResponseDto<StartProcessResponse>>
    {
        public async Task<BaseResponseDto<StartProcessResponse>> Handle(
            StartProcessITSMCommand command,
            CancellationToken ct)
        {
            // 0) Процесс
            var process = await unitOfWork.ProcessRepository
                             .GetByFilterAsync(ct, p => p.ProcessCode == command.ProcessCode)
                         ?? throw new HandlerException($"Процесс с кодом {command.ProcessCode} не найден",
                             ErrorCodesEnum.Business);

            // 1) Готовим payload JSON (как есть, максимум — впишем regnum и startdate если их нет)
            var jsonOptions = new JsonSerializerOptions { Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping };
            var payloadJson = JsonSerializer.Serialize(command.Payload ?? new Dictionary<string, object>(), jsonOptions);

            // вытащим заголовок документа (если есть)
            var documentTitle = TryRead<string>(payloadJson, "processData", "documentTitle");

            // если regnum не задан — сгенерируем и впишем
            var regnum = TryRead<string>(payloadJson, "regData", "regnum");
            if (string.IsNullOrWhiteSpace(regnum))
            {
                regnum = await helperService.GenerateRequestNumberAsync(command.ProcessCode, ct);
                payloadJson = SetInPayload(payloadJson, jsonOptions, ("regData", "regnum", regnum));
            }

            // всегда проставим startdate (момент старта)
            payloadJson = SetInPayload(payloadJson, jsonOptions, ("regData", "startdate", DateTime.Now.ToString("o")));

            // 2) Создаём ProcessData = Started
            var created = new ProcessDataCreatedEvent
            {
                ProcessId     = process.Id,
                ProcessCode   = process.ProcessCode,
                ProcessName   = process.ProcessName,
                RegNumber     = regnum,
                InitiatorCode = command.InitiatorCode,
                InitiatorName = command.InitiatorName,
                StatusCode    = "Started",
                StatusName    = "В работе",
                PayloadJson   = payloadJson,
                Title         = documentTitle,
                StartedAt     = DateTime.Now
            };
            await unitOfWork.ProcessDataRepository.RaiseEvent(created, ct);

            // 3) Стартуем Camunda и связываем
            var pi = await camundaService.CamundaStartProcess(new CamundaStartProcessRequest
            {
                processCode = command.ProcessCode,
                variables   = new Dictionary<string, object> { { "processGuid", created.EntityId.ToString() } }
            });

            await unitOfWork.ProcessDataRepository.RaiseEvent(
                new ProcessDataProcessInstanseIdChangedEvent
                {
                    EntityId          = created.EntityId,
                    ProcessInstanceId = pi
                },
                ct);

            // 4) Апсертим files из payload (если есть)
            await UpsertFilesFromPayloadAsync(created.EntityId, payloadJson, unitOfWork, ct);

            // 5) Пишем историю "Start"
            await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(new ProcessTaskHistoryCreatedEvent
            {
                ProcessDataId = created.EntityId,
                TaskId        = created.EntityId,
                Action        = "Start",
                ActionName    = "Зарегистрировано",
                BlockName     = "Регистрационная форма",
                Timestamp     = DateTime.Now,
                PayloadJson   = payloadJson,
                Comment       = "",
                Description   = "",

                // важные not null
                ProcessCode   = process.ProcessCode,
                ProcessName   = process.ProcessName,
                RegNumber     = regnum,
                InitiatorCode = command.InitiatorCode,
                InitiatorName = command.InitiatorName,
                AssigneeCode  = command.InitiatorCode,
                AssigneeName  = command.InitiatorName,
                Title         = documentTitle
            }, ct);

            return new BaseResponseDto<StartProcessResponse>
            {
                Data = new StartProcessResponse
                {
                    ProcessGuid = created.EntityId,
                    RegNumber   = regnum
                }
            };
        }

        // ===== helpers =====

        private static T? TryRead<T>(string json, params string[] path)
        {
            try
            {
                var node = JsonNode.Parse(json);
                foreach (var p in path)
                {
                    node = (node as JsonObject)?[p];
                    if (node is null) return default;
                }

                return node.Deserialize<T>(new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
            }
            catch { return default; }
        }

        private static string SetInPayload(string json, JsonSerializerOptions opts, (string section, string key, object value) kv)
        {
            var root = (JsonNode.Parse(json) as JsonObject) ?? new JsonObject();

            if (root[kv.section] is not JsonObject sec)
            {
                sec = new JsonObject();
                root[kv.section] = sec;
            }

            sec[kv.key] = JsonSerializer.SerializeToNode(kv.value, kv.value.GetType(), opts);
            return JsonSerializer.Serialize(root, opts);
        }

        private static async Task UpsertFilesFromPayloadAsync(
            Guid processDataId,
            string payloadJson,
            IUnitOfWork unitOfWork,
            CancellationToken ct)
        {
            try
            {
                var root  = JsonNode.Parse(payloadJson)!.AsObject();
                var files = root["files"] as JsonArray;
                if (files is null || files.Count == 0) return;

                // берём всё (и удалённые, и активные)
                var existing = await unitOfWork.ProcessFileRepository.GetByFilterListAsync(
                    ct, f => f.ProcessDataId == processDataId);

                var byFileId = existing
                    .Where(x => x.FileId != Guid.Empty)
                    .ToDictionary(x => x.FileId, x => x);

                foreach (var node in files.OfType<JsonObject>())
                {
                    var fileIdStr = node["fileId"]?.GetValue<string>();
                    var fileName  = node["fileName"]?.GetValue<string>();
                    var fileType  = node["fileType"]?.GetValue<string>();

                    if (string.IsNullOrWhiteSpace(fileName) || string.IsNullOrWhiteSpace(fileType))
                        throw new HandlerException("Каждый объект в \"files\" должен иметь непустые fileName и fileType",
                            ErrorCodesEnum.Business);

                    if (string.IsNullOrWhiteSpace(fileIdStr) || !Guid.TryParse(fileIdStr, out var fileId))
                        throw new HandlerException("Каждый объект в \"files\" должен иметь корректный GUID в поле fileId",
                            ErrorCodesEnum.Business);

                    if (!byFileId.TryGetValue(fileId, out var ex))
                    {
                        await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileCreatedEvent
                        {
                            EntityId      = Guid.NewGuid(),
                            ProcessDataId = processDataId,
                            FileId        = fileId,
                            FileName      = fileName!,
                            FileType      = fileType!
                        }, ct);
                    }
                    else
                    {
                        if (ex.State != ProcessFileState.Active || ex.FileName != fileName || ex.FileType != fileType)
                        {
                            await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileStatusChangedEvent
                            {
                                EntityId = ex.Id,
                                State    = ProcessFileState.Active,
                                FileName = fileName,
                                FileType = fileType
                            }, ct);
                        }
                    }
                }
            }
            catch
            {
                // опционально: логирование
            }
        }
    }
}


{
  "processCode": "ChangeRequest",
  "initiatorCode": "m.ilespayev",
  "initiatorName": "string",
  "payload": {
   
 
  "regData": {
    "userCode": "m.ilespayev",
    "userName": "Илеспаев Меииржан Анварович",
    "departmentId": "00ЗП-0013",
    "departmentName": "Управление автоматизации бизнес-процессов",
    "startdate": "2025-09-22T14:18:22.3148923+05:00",
    "regnum": ""
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
    "loginAD": "m.ilespayev",
    "shortName": "Илеспаев М.А."
  },
  "processData": {
    "CustomerId": "00ЗП-0010",
    "CustomerName": "Департамент цифровой трансформации",
    "DocumentTitle": "erwerwer",
    "SystemId": "11111111-0000-0000-0000-000000000002,11111111-0000-0000-0000-000000000003",
    "SystemName": "1С Предприятие 8, модуль Бюджетирование,1С Предприятие 8, модуль Учет договоров",
    "ProcessCategoryId": "",
    "ProcessCategoryName": "",
    "ProcessesId": "",
    "ProcessesName": "",
    "SystemOwnerId": "00ЗП-0010",
    "SystemOwnerName": "",
    "ChangeReason": "fdsfsdf",
    "PlanningInfo": "",
    "CurrentFunctionality": "sdffd",
    "Requirements": "32424",
    "Impact": "geg",
    "TestCases": "ergerger"
  }
 
 
  }
}

{
  "data": null,
  "message": "Некорректный ответ от Camunda: Response status code does not indicate success: 500 (). ;",
  "errorCode": 1001
}


