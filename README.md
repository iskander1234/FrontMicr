{
  "taskId": "9f322c76-5747-4954-ade8-720c878f1960",
  "action": "Submit",
  "condition": "accept",
  "senderUserCode": "string",
  "senderUserName": "string",
  "comment": "string",
  "payloadJson": {
    "regData": {
    "userCode": "m.ilespayev",
    "userName": "Илеспаев Меииржан Анварович",
    "departmentId": "00ЗП-0013",
    "departmentName": "Управление автоматизации бизнес-процессов",
    "startdate": "2025-09-22T19:58:22.7125454+05:00",
    "regnum": "CH-0040-2025"
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
    "analyst":
    {
      "classificationCode": "Normal",
      "classificationName": "Нормальное изменение ",
      "deadline": "5"
    },
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

using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Queries.ITSM;
using BpmBaseApi.Shared.Queries.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using Microsoft.AspNetCore.Mvc;
using Swashbuckle.AspNetCore.Annotations;

namespace BpmBaseApi.Controllers.V1
{
    [Route("api/v{version:apiVersion}/[controller]")]
    [ApiController]
    [SwaggerTag("Управление этапами процесса ITSM")]
    public class ITSMController(IMediator mediator) : ControllerBase
    {
        /// <summary>
        /// Получение список задач по blockCode
        /// </summary>
        /// <param name="command">Название этапа и логин пользователя</param>
        /// <param name="cancellationToken">Маркер отмены, используемый для отмены HTTP-запроса.</param>
        /// <returns></returns>
        [HttpPost("tasks")]
        [SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<List<GetUserTasksResponse>>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        [SwaggerResponse(StatusCodes.Status503ServiceUnavailable)]
        public async Task<IActionResult> GetUserTasksByBlockCode(GetITSMUserTasksQuery command, CancellationToken cancellationToken)
        {
            var result = await mediator.Send(command, cancellationToken);

            return Ok(result);
        }
        
        [HttpPost("start")]
        [SwaggerResponse(StatusCodes.Status200OK, "Процесс запущен", typeof(BaseResponseDto<StartProcessResponse>))]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        public async Task<IActionResult> StartItsmAsync([FromBody] StartProcessITSMCommand command, CancellationToken ct)
        {
            var result = await mediator.Send(command, ct);
            return Ok(result);
        }
        
        [HttpPost("send")]
        [SwaggerResponse(StatusCodes.Status200OK, "Отправлено", typeof(BaseResponseDto<SendProcessResponse>))]
        public async Task<IActionResult> SendItsmAsync([FromBody] SendProcessCommand command, CancellationToken ct)
        {
            var result = await mediator.Send(command, ct);
            return Ok(result);
        }
    }
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
    /// - читает classification из payloadJson (processData.analyst.classificationCode | classification | level | priority)
    /// - при системном parent: шлёт ВМЕСТЕ со stage-булевыми переменными (как старый Send) => gateway видит ${classification == 'Normal'}
    /// - иначе поднимаем родителя в Pending
    /// </summary>
    public class SendProcessITSMCommandHandler(
        IUnitOfWork unitOfWork,
        ICamundaService camundaService,
        IProcessTaskService processTaskService)
        : IRequestHandler<SendProcessCommand, BaseResponseDto<SendProcessResponse>>
    {
        public async Task<BaseResponseDto<SendProcessResponse>> Handle(
            SendProcessCommand command, CancellationToken ct)
        {
            // Текущая подзадача должна быть Pending
            var currentTask = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                                  ct, t => t.Id == command.TaskId && t.Status == "Pending")
                              ?? throw new HandlerException("Задача не найдена или уже обработана",
                                  ErrorCodesEnum.Business);

            var processData = await unitOfWork.ProcessDataRepository.GetByIdAsync(ct, currentTask.ProcessDataId)
                              ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

            var parentTask = await unitOfWork.ProcessTaskRepository.GetByIdAsync(ct, currentTask.ParentTaskId!.Value);

            if (string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
            {
                var claimed = await camundaService.CamundaClaimedTasks(
                                  processData.ProcessInstanceId, parentTask.BlockCode)
                              ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);

                // 1) булевые для этапа (как в старом Send)
                var stage = Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var st) ? st : ProcessStage.Approval;
                var stageVars = await BuildCamundaVariablesAsync(
                    stage, command.Condition, command.Action,
                    processData.Id, parentTask.Id, unitOfWork, ct);

                // 2) строковая classification
                var classVars = BuildClassificationVariables(command.PayloadJson);

                // 3) объединяем
                foreach (var kv in classVars) stageVars[kv.Key] = kv.Value;

                var submit = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
                {
                    TaskId = claimed.TaskId,
                    Variables = stageVars
                });

                if (!submit.Success)
                    throw new HandlerException($"Ошибка при отправке Msg:{submit.Msg}", ErrorCodesEnum.Camunda);

                await processTaskService.FinalizeTaskAsync(parentTask, ct);
            }
            else
            {
                // Родитель не system — делаем его Pending
                await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
                {
                    EntityId = parentTask.Id,
                    Status = "Pending"
                }, ct);

                if (!string.IsNullOrWhiteSpace(parentTask.AssigneeCode))
                    await processTaskService.RefreshUserTaskCacheAsync(parentTask.AssigneeCode, ct);
            }

            // Лог + закрытие текущей подзадачи
            await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, ct);
            await processTaskService.FinalizeTaskAsync(currentTask, ct);
            await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, ct);

            return new BaseResponseDto<SendProcessResponse>
            {
                Data = new SendProcessResponse { Success = true }
            };
        }

        /// <summary>
        /// Ищем classification в payload (вложенность поддерживается).
        /// Возвращаем {"classification": "Emergency|Standard|Normal"} либо пустой словарь.
        /// </summary>
        private static Dictionary<string, object> BuildClassificationVariables(Dictionary<string, object>? payload)
        {
            if (payload is null) return new();

            static string? ReadStr(Dictionary<string, object> p, string key)
            {
                if (!p.TryGetValue(key, out var v) || v is null) return null;
                var json = JsonSerializer.Serialize(v);
                return JsonSerializer.Deserialize<string>(json,
                    new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
            }

            // Ищем processData.analyst.classificationCode
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
                    if (analyst != null) fromAnalyst = ReadStr(analyst, "classificationCode");
                }
            }

            var level = fromAnalyst
                        ?? ReadStr(payload, "classification")
                        ?? ReadStr(payload, "itsmLevel")
                        ?? ReadStr(payload, "level")
                        ?? ReadStr(payload, "priority");

            if (string.IsNullOrWhiteSpace(level)) return new();

            var canonical = level.Trim().ToLowerInvariant() switch
            {
                "emergency" => "Emergency",
                "standard"  => "Standard",
                "normal"    => "Normal",
                _ => null
            };

            return canonical is null ? new() : new() { ["classification"] = canonical };
        }

        // ==== Вспомогалки для этапных булевых, как в старом Send ====

        private static string GetCamundaVarNameForStage(ProcessStage stage) => stage switch
        {
            ProcessStage.Approval       => "agreement",
            ProcessStage.Signing        => "sign",
            ProcessStage.Execution      => "executed",
            ProcessStage.ExecutionCheck => "executionCheck",
            ProcessStage.Rework         => "refinement",
            ProcessStage.Registration   => "registrar",
            _                           => "agreement"
        };

        private static async Task<Dictionary<string, object>> BuildCamundaVariablesAsync(
            ProcessStage stage,
            ProcessCondition condition,
            ProcessAction action,
            Guid processDataId,
            Guid? parentTaskId,
            IUnitOfWork unitOfWork,
            CancellationToken ct)
        {
            static bool? FromCondition(ProcessCondition c) => c switch
            {
                ProcessCondition.accept => true,
                ProcessCondition.remake or ProcessCondition.reject => false,
                _ => (bool?)null
            };

            static bool? FromAction(ProcessAction a) => a switch
            {
                ProcessAction.Submit => true,
                ProcessAction.Reject => false,
                ProcessAction.Cancel => false,
                _ => (bool?)null
            };

            // В Execution в BPMN нет условия — переменные не шлём вовсе.
            if (stage == ProcessStage.Execution)
                return new Dictionary<string, object>();

            var varName = GetCamundaVarNameForStage(stage);

            bool conditionDriven = stage is
                ProcessStage.Approval
                or ProcessStage.Signing
                or ProcessStage.ExecutionCheck
                or ProcessStage.Rework
                or ProcessStage.Registration;

            bool isPositive;

            if (conditionDriven)
            {
                var byCond = FromCondition(condition);
                var byAct  = FromAction(action);
                isPositive = byCond ?? byAct ?? false;

                if (stage == ProcessStage.Approval && isPositive)
                {
                    isPositive = await IsStagePositiveConsideringHistoryAsync(
                        ProcessStage.Approval, isPositive, processDataId, parentTaskId, unitOfWork, ct);
                }
            }
            else
            {
                isPositive = FromAction(action) ?? false;
            }

            return new Dictionary<string, object> { [varName] = isPositive };
        }

        private static async Task<bool> IsStagePositiveConsideringHistoryAsync(
            ProcessStage stage,
            bool currentIsPositive,
            Guid processDataId,
            Guid? parentTaskId,
            IUnitOfWork unitOfWork,
            CancellationToken ct)
        {
            if (!currentIsPositive) return false;

            var stageCode = stage.ToString();

            var negativeCount = await unitOfWork.ProcessTaskHistoryRepository.CountAsync(
                ct,
                h => h.ProcessDataId == processDataId
                     && h.BlockCode == stageCode
                     && h.ParentTaskId == parentTaskId
                     && (h.Condition == nameof(ProcessCondition.remake)
                         || h.Condition == nameof(ProcessCondition.reject)));

            return negativeCount == 0;
        }
    }
}



Code	Details
200	
Response body
Download
{
  "data": null,
  "message": "Ошибка при отправке Msg:Error submitting task: Unknown property used in expression: ${classification == 'Normal'}. Cause: Cannot resolve identifier 'classification'",
  "errorCode": 1002
}

