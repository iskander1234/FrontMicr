запрос такой {
    "taskId": "aa6fb946-7f7e-470e-b506-7930d85668f3",
    "action": "Submit",
    "condition": "accept",
    "senderUserCode": "r.mukhamatzhanov",
    "senderUserName": "Мухаматжанов Рамазан Рустамжанович",
    "comment": "comment",
    "payloadJson": {
        "processCode": "ServiceRequest",
        "initiatorCode": "r.mukhamatzhanov",
        "initiatorName": "Мухаматжанов Рамазан Рустамжанович",
        "payload": {
            "regData": {
                "userCode": "r.mukhamatzhanov",
                "userName": "Мухаматжанов Рамазан Рустамжанович",
                "departmentId": "00ЗП-0013",
                "departmentName": "Управление автоматизации бизнес-процессов",
                "startdate": "2025-09-23T09:41:00.289Z",
                "regnum": ""
            },
            "sysInfo": {
                "userCode": "r.mukhamatzhanov",
                "userName": "Мухаматжанов Рамазан Рустамжанович",
                "comment": "comment",
                "action": "submit",
                "condition": "string"
            },
            "initiator": {
                "id": 1609,
                "name": "Мухаматжанов Рамазан Рустамжанович",
                "shortName": "Мухаматжанов Р.Р.",
                "position": "Главный специалист",
                "login": "r.mukhamatzhanov",
                "statusCode": 6,
                "statusDescription": "Работа",
                "depId": "00ЗП-0013",
                "depName": "Управление автоматизации бизнес-процессов",
                "parentDepId": "00ЗП-0010",
                "parentDepName": "Департамент цифровой трансформации",
                "isFilial": false,
                "mail": "r.mukhamatzhanov@enpf.kz",
                "localPhone": "0",
                "mobilePhone": "+7(777) 493-73-31",
                "isManager": false,
                "managerTabNumber": "4340",
                "disabled": false,
                "tabNumber": "00ЗП-00360",
                "loginAD": "r.mukhamatzhanov"
            },
            "processData": {
                "id": "",
                "documentTitle": "Компьютерное оборудование",
                "documentBody": "",
                "savedData": [
                    {
                        "subjectId": "af152f4b-7bfc-4f49-92d5-7c5df0ea68ef",
                        "subjectLabel": "Компьютерное оборудование",
                        "subjectLevel": 1,
                        "contents": []
                    },
                    {
                        "subjectId": "1fb8043d-a47f-4ace-adf1-1299341c213b",
                        "subjectLabel": "Картридеры",
                        "contents": [],
                        "subjectLevel": 2
                    },
                    {
                        "subjectId": "bd372353-bb2d-4b0d-b1e6-d7f407b916fe",
                        "subjectLabel": "Отсутствует связь с ПК",
                        "contents": [
                            {
                                "question": "В чём проявляется проблема",
                                "type": "textarea",
                                "id": "bd83ff17-83c0-4875-8992-1b2c21f0b005",
                                "subjectId": "bd372353-bb2d-4b0d-b1e6-d7f407b916fe",
                                "values": []
                            }
                        ],
                        "subjectLevel": 3
                    }
                ],
                "tempContents": [
                    {
                        "question": "В чём проявляется проблема",
                        "type": "textarea",
                        "id": "bd83ff17-83c0-4875-8992-1b2c21f0b005",
                        "subjectId": "bd372353-bb2d-4b0d-b1e6-d7f407b916fe",
                        "values": []
                    }
                ],
                "selectedOptions": [
                    "qweqweqwe"
                ],
                "level1": {
                    "formData": {
                        "feature": "problem",
                        "comment": "comment",
                        "criticality": "green",
                        "execution": "toSecondLine",
                        "priority": "first"
                    },
                    "usr": {
                        "userCode": "r.mukhamatzhanov",
                        "userName": "Мухаматжанов Рамазан Рустамжанович",
                        "departmentId": "00ЗП-0013",
                        "departmentName": "Управление автоматизации бизнес-процессов",
                        "date": "2025-09-23T09:41:00.289Z"
                    }
                }
            },
            "files": []
        }
    }
}

Ошибка {
  "data": null,
  "message": "Не найдена задача в Camunda: f7560e61-986b-11f0-857a-0242ac160004 Level1;",
  "errorCode": 1001
}


using System.Text.Json;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Models.Camunda;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Shared.Enum;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    /// <summary>
    /// ITSM Send:
    /// - читает classification из payloadJson (processData.analyst.classificationCode | classification | itsmLevel | level | priority)
    /// - при системном parent: отправляет в Camunda submit с переменной classification
    /// - иначе поднимает родителя в Pending
    /// </summary>
    public class SendProcessITSMCommandHandler(
        IUnitOfWork unitOfWork,
        ICamundaService camundaService,
        IProcessTaskService processTaskService)
        : IRequestHandler<SendItsmProcessCommand, BaseResponseDto<SendProcessResponse>>
    {
        public async Task<BaseResponseDto<SendProcessResponse>> Handle(
            SendItsmProcessCommand command, CancellationToken ct)
        {
            // 1) Текущая подзадача должна быть Pending
            var currentTask = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                                  ct, t => t.Id == command.TaskId && t.Status == "Pending")
                              ?? throw new HandlerException("Задача не найдена или уже обработана",
                                  ErrorCodesEnum.Business);

            var processData = await unitOfWork.ProcessDataRepository.GetByIdAsync(ct, currentTask.ProcessDataId)
                              ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

            var parentTask = await unitOfWork.ProcessTaskRepository.GetByIdAsync(ct, currentTask.ParentTaskId!.Value);

            // 2) Читаем и нормализуем classification
          //  var classification = ReadClassification(command.PayloadJson);
            // if (string.IsNullOrWhiteSpace(classification))
            //     throw new HandlerException(
            //         "Не удалось определить classification (ожидается Emergency|Standard|Normal).",
            //         ErrorCodesEnum.Business);

            if (string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
            {
                var claimed = await camundaService.CamundaClaimedTasks(
                                  processData.ProcessInstanceId, parentTask.BlockCode)
                              ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);

                // Больше НЕ требуем classification:
                var variables = new Dictionary<string, object>();

// Если вдруг пришла классификация — можно добавить, но это НЕ обязательно
                var classification = ReadClassification(command.PayloadJson);
                if (!string.IsNullOrWhiteSpace(classification))
                {
                    variables["classification"] = classification;
                }

// Логика для Level1/Level2/Level3
                if (Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var stage))
                {
                    switch (stage)
                    {
                        case ProcessStage.Level1:
                        {
                            // executed => true, toSecondLine => false
                            var execution = ReadExecution(command.PayloadJson); // "executed" | "toSecondLine" | null
                            bool line1 = string.Equals(execution, "executed", StringComparison.OrdinalIgnoreCase)
                                ? true
                                : string.Equals(execution, "toSecondLine", StringComparison.OrdinalIgnoreCase)
                                    ? false
                                    : true; // дефолт если не пришло - true
                            variables["Line 1"] = line1;
                            break;
                        }
                        case ProcessStage.Level2:
                        {
                            bool line2 = ReadLineBool(command.PayloadJson, 2) ?? true;
                            variables["Line 2"] = line2;
                            break;
                        }
                        case ProcessStage.Level3:
                        {
                            bool line3 = ReadLineBool(command.PayloadJson, 3) ?? true;
                            variables["Line 3"] = line3;
                            break;
                        }
                    }
                }

                var submit = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
                {
                    TaskId    = claimed.TaskId,
                    Variables = variables
                });

                if (!submit.Success)
                    throw new HandlerException($"Ошибка при отправке Msg:{submit.Msg}", ErrorCodesEnum.Camunda);

                // <-- ВАЖНО: сначала лог и финализация текущей (дочерней) задачи
                await processTaskService.LogHistoryItsmAsync(currentTask, command, currentTask.AssigneeCode, ct);
                await processTaskService.FinalizeTaskAsync(currentTask, ct);
                await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, ct);

                // <-- Затем финализируем родителя (system)
                await processTaskService.FinalizeTaskAsync(parentTask, ct);
            }
            else
            {
                await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
                {
                    EntityId = parentTask.Id,
                    Status = "Pending"
                }, ct);

                if (!string.IsNullOrWhiteSpace(parentTask.AssigneeCode))
                    await processTaskService.RefreshUserTaskCacheAsync(parentTask.AssigneeCode, ct);

                await processTaskService.LogHistoryItsmAsync(currentTask, command, currentTask.AssigneeCode, ct);
                await processTaskService.FinalizeTaskAsync(currentTask, ct);
                await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, ct);
            }

            // 5) Лог + закрытие текущей подзадачи
            await processTaskService.LogHistoryItsmAsync(currentTask, command, currentTask.AssigneeCode, ct);
            await processTaskService.FinalizeTaskAsync(currentTask, ct);
            await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, ct);

            return new BaseResponseDto<SendProcessResponse>
            {
                Data = new SendProcessResponse { Success = true }
            };
        }

        /// <summary>
        /// Возвращает каноническое значение "Emergency" | "Standard" | "Normal"
        /// из payload: processData.analyst.classificationCode | classification | itsmLevel | level | priority.
        /// </summary>
        private static string? ReadClassification(Dictionary<string, object>? payload)
        {
            if (payload is null) return null;

            static string? ReadStr(Dictionary<string, object> p, string key)
            {
                if (!p.TryGetValue(key, out var v) || v is null) return null;
                var json = JsonSerializer.Serialize(v);
                return JsonSerializer.Deserialize<string>(json,
                    new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
            }

            string? fromAnalyst = null;
            if (payload.TryGetValue("processData", out var pdRaw) && pdRaw is not null)
            {
                var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(
                    JsonSerializer.Serialize(pdRaw), new JsonSerializerOptions { PropertyNameCaseInsensitive = true });

                if (pd != null && pd.TryGetValue("analyst", out var aRaw) && aRaw is not null)
                {
                    var analyst = JsonSerializer.Deserialize<Dictionary<string, object>>(
                        JsonSerializer.Serialize(aRaw),
                        new JsonSerializerOptions { PropertyNameCaseInsensitive = true });

                    if (analyst != null)
                        fromAnalyst = ReadStr(analyst, "classificationCode");
                }
            }

            var raw = fromAnalyst
                      ?? ReadStr(payload, "classification")
                      ?? ReadStr(payload, "itsmLevel")
                      ?? ReadStr(payload, "level")
                      ?? ReadStr(payload, "priority");

            if (string.IsNullOrWhiteSpace(raw)) return null;

            var n = raw.Trim().ToLowerInvariant();
            return n switch
            {
                "emergency" => "Emergency",
                "standard" => "Standard",
                "normal" => "Normal",
                _ => null
            };
        }

        private static string? ReadExecution(Dictionary<string, object>? payload)
        {
            // ищем: payload -> processData -> level1 -> formData -> execution
            try
            {
                if (payload is null) return null;

                if (!payload.TryGetValue("payload", out var pRaw) || pRaw is null) return null;
                var p = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pRaw));

                if (p is null || !p.TryGetValue("processData", out var pdRaw) || pdRaw is null) return null;
                var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pdRaw));

                if (pd is null || !pd.TryGetValue("level1", out var lRaw) || lRaw is null) return null;
                var level1 = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(lRaw));

                if (level1 is null || !level1.TryGetValue("formData", out var fdRaw) || fdRaw is null) return null;
                var form = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(fdRaw));

                if (form is null || !form.TryGetValue("execution", out var exRaw) || exRaw is null) return null;
                return JsonSerializer.Deserialize<string>(JsonSerializer.Serialize(exRaw));
            }
            catch
            {
                return null;
            }
        }

        private static bool? ReadLineBool(Dictionary<string, object>? payload, int levelIndex)
        {
            // Level2: payload->processData->level2->formData->line2 (bool)
            // Level3: payload->processData->level3->formData->line3 (bool)
            try
            {
                if (payload is null) return null;

                if (!payload.TryGetValue("payload", out var pRaw) || pRaw is null) return null;
                var p = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pRaw));

                if (p is null || !p.TryGetValue("processData", out var pdRaw) || pdRaw is null) return null;
                var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(pdRaw));

                var levelKey = $"level{levelIndex}";
                if (pd is null || !pd.TryGetValue(levelKey, out var lRaw) || lRaw is null) return null;
                var level = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(lRaw));

                if (level is null || !level.TryGetValue("formData", out var fdRaw) || fdRaw is null) return null;
                var form = JsonSerializer.Deserialize<Dictionary<string, object>>(JsonSerializer.Serialize(fdRaw));

                var lineKey = $"line{levelIndex}";
                if (form is null || !form.TryGetValue(lineKey, out var lineRaw) || lineRaw is null) return null;

                return JsonSerializer.Deserialize<bool?>(JsonSerializer.Serialize(lineRaw));
            }
            catch
            {
                return null;
            }
        }
    }
}
