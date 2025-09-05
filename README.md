В начале он правильно от сортирован но после каких то операций сортировка ломается GetUserTasksByStageBaseHandler но я сортирую по Created но думаю что кеш както может по ломать мою сортировку по смотри пожалуйста  using AutoMapper;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using Microsoft.Extensions.Caching.Memory;

namespace BpmBaseApi.Application.QueryHandlers.Process;

public abstract class GetUserTasksByStageBaseHandler<TRequest>
    : IRequestHandler<TRequest, BaseResponseDto<List<GetUserTasksResponse>>>
    where TRequest : IRequest<BaseResponseDto<List<GetUserTasksResponse>>>
{
    private readonly IMapper _mapper;
    private readonly IUnitOfWork _uow;
    private readonly IMemoryCache _cache;
    private readonly string _stageCode; // "Approval" | "Signing" | "Execution" | "ExecutionCheck"

    protected GetUserTasksByStageBaseHandler(IMapper mapper, IUnitOfWork uow, IMemoryCache cache, string stageCode)
    { _mapper = mapper; _uow = uow; _cache = cache; _stageCode = stageCode; }

    private static string GetUserCode(TRequest req)
        => (string?)typeof(TRequest).GetProperty("UserCode")?.GetValue(req)
           ?? throw new InvalidOperationException("UserCode is required");

    public async Task<BaseResponseDto<List<GetUserTasksResponse>>> Handle(TRequest request, CancellationToken ct)
    {
        var userCode = GetUserCode(request);
        var userLower = userCode.ToLowerInvariant();
        var cacheKey = $"tasks:{_stageCode}:{userLower}";

        if (_cache.TryGetValue(cacheKey, out List<GetUserTasksResponse> cached))
            return new() { Data = cached };

        // СВОИ Pending по нужному этапу
        var own = await _uow.ProcessTaskRepository.GetByFilterListAsync(
            ct,
            p => p.AssigneeCode != null
                 && p.AssigneeCode.ToLower() == userLower
                 && p.Status == "Pending"
                 && p.BlockCode == _stageCode   // <-- ВАЖНО: без StringComparison
        );

        var all = new List<ProcessTaskEntity>(own);

        // ЗАМЕСТИТЕЛЬ
        var asDeputy = await _uow.DelegationRepository.GetByFilterListAsync(
            ct, d => d.DeputyUserCode != null && d.DeputyUserCode.ToLower() == userLower);

        if (asDeputy.Any())
        {
            var principals = asDeputy
                .Where(d => d.PrincipalUserCode != null)
                .Select(d => d.PrincipalUserCode!.ToLower())
                .Distinct()
                .ToList();

            var principalTasks = await _uow.ProcessTaskRepository.GetByFilterListAsync(
                ct,
                p => p.AssigneeCode != null
                     && principals.Contains(p.AssigneeCode.ToLower())
                     && p.Status == "Pending"
                     && p.BlockCode == _stageCode
            );

            all.AddRange(principalTasks);
        }

        // сортировка (новые сверху)
        var ordered = all
            .GroupBy(t => t.Id)            // убрать дубликаты
            .Select(g => g.First())
            .OrderByDescending(t => t.Created)
            .ToList();

        var response = _mapper.Map<List<GetUserTasksResponse>>(ordered);
        _cache.Set(cacheKey, response, TimeSpan.FromHours(1));
        return new() { Data = response };
    }
}


Это чтобы ты аолнял как работает кеш и где 

public async Task FinalizeTaskAsync(ProcessTaskEntity task, CancellationToken cancellationToken)
        {
            await unitOfWork.ProcessTaskRepository.DeleteByIdAsync(task.Id, cancellationToken);

            // безопасно: null / пробелы игнорим
            if (!string.IsNullOrWhiteSpace(task.AssigneeCode))
                await RefreshUserTaskCacheAsync(task.AssigneeCode!, cancellationToken);
        }

        private static string NormalizeUser(string userCode)
            => userCode.Trim().ToLowerInvariant();

        private static string UserRootKey(string userNorm)
            => $"tasks:{userNorm}";

        private static string UserStagesIndexKey(string userNorm)
            => $"tasks:{userNorm}:__stages";

        private static string StageKey(string stageCode, string userNorm)
            => $"tasks:{stageCode}:{userNorm}";

        public async Task RefreshUserTaskCacheAsync(string userCode, CancellationToken cancellationToken)
        {
            if (string.IsNullOrWhiteSpace(userCode)) return;

            var userNorm = NormalizeUser(userCode);

            // 1) Снести прошлые ключи, чтобы не висели «пустые стадии»
            cache.Remove(UserRootKey(userNorm));
            if (cache.TryGetValue<string[]>(UserStagesIndexKey(userNorm), out var oldStageKeys) &&
                oldStageKeys is not null)
            {
                foreach (var k in oldStageKeys) cache.Remove(k);
                cache.Remove(UserStagesIndexKey(userNorm));
            }

            // 2) Прочитать актуальное Pending (защита от null в БД + case-insensitive)
            var updatedTasks = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
                cancellationToken,
                p => p.AssigneeCode != null
                     && p.Status == "Pending"
                     && p.AssigneeCode.ToLower() == userNorm
            );

            var mappedAll = mapper.Map<List<GetUserTasksResponse>>(updatedTasks);

            // TTL лучше короткий, но чтобы «не ломать» — вынесем в константу (можешь поставить minutes:1)
            var ttl = TimeSpan.FromMinutes(1);

            // 3) Положить общий ключ
            cache.Set(UserRootKey(userNorm), mappedAll, ttl);

            // 4) Разложить по стадиям + сохранить индекс стадий, чтобы в следующий раз удалить корректно
            var newStageKeys = new List<string>();
            foreach (var grp in updatedTasks.GroupBy(t => t.BlockCode))
            {
                var stageKey = StageKey(grp.Key, userNorm);
                var mappedStage = mapper.Map<List<GetUserTasksResponse>>(grp.ToList());
                cache.Set(stageKey, mappedStage, ttl);
                newStageKeys.Add(stageKey);
            }

            cache.Set(UserStagesIndexKey(userNorm), newStageKeys.ToArray(), ttl);
        }
        
        using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Enum;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using System.Text.Json;
using AutoMapper;
using Microsoft.Extensions.Caching.Memory;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Models.Process;
using BpmBaseApi.Shared.Models.Camunda;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    public class SendProcessCommandHandler(
        IMapper mapper,
        IUnitOfWork unitOfWork,
        IMemoryCache cache,
        ICamundaService camundaService,
        IProcessTaskService processTaskService)
        : IRequestHandler<SendProcessCommand, BaseResponseDto<SendProcessResponse>>
    {
        public async Task<BaseResponseDto<SendProcessResponse>> Handle(SendProcessCommand command,
            CancellationToken cancellationToken)
        {
            // Берём задачу без фильтра по статусу — чтобы поймать Waiting и вернуть 400 с подсказкой
            var currentTask = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                cancellationToken,
                p => p.Id == command.TaskId
            ) ?? throw new HandlerException("Задача не найдена", ErrorCodesEnum.Business);

            var processData =
                await unitOfWork.ProcessDataRepository.GetByIdAsync(cancellationToken, currentTask.ProcessDataId)
                ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

            // Последовательный ли режим из processData.approvalTypeCode
            var pdPayload = JsonSerializer.Deserialize<Dictionary<string, object>>(processData.PayloadJson);
            bool isSequential = ReadIsSequentialFromProcessData(pdPayload);

            // Если Sequentially и задача НЕ Pending → 400 и сказать кто Pending
            if (isSequential && !string.Equals(currentTask.Status, "Pending", StringComparison.OrdinalIgnoreCase))
            {
                ProcessTaskEntity? activePending = null;

                if (currentTask.ParentTaskId != null)
                {
                    activePending = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                        cancellationToken,
                        t => t.ParentTaskId == currentTask.ParentTaskId && t.Status == "Pending"
                    );
                }

                if (activePending == null)
                {
                    activePending = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                        cancellationToken,
                        t => t.ProcessDataId == currentTask.ProcessDataId
                             && t.BlockCode == currentTask.BlockCode
                             && t.Status == "Pending"
                    );
                }

                var responsible = activePending?.AssigneeCode ?? "не найден";
                throw new HandlerException(
                    $"Отправка запрещена: сначала должен отправить исполнитель со статусом Pending ({responsible}).",
                    ErrorCodesEnum.Business // у тебя это маппится в HTTP 400
                );
            }

            // Старая проверка «только Pending» (для параллельного режима как было)
            if (!string.Equals(currentTask.Status, "Pending", StringComparison.OrdinalIgnoreCase))
                throw new HandlerException("Задача не найдена или уже обработана", ErrorCodesEnum.Business);

            var recipients = ExtractFromPayload<List<UserInfo>>(command.PayloadJson, "recipients") ?? new();
            //var comment = ExtractFromPayload<string>(command.PayloadJson, "comment");

            // Делегирование
            if (command.Action == ProcessAction.Delegate && recipients.Any())
            {
                await processTaskService.HandleDelegationAsync(currentTask, processData, command, recipients,
                    cancellationToken);
                return new BaseResponseDto<SendProcessResponse>
                {
                    Data = new SendProcessResponse { Success = true }
                };
            }

            // Если sequential — закрываем текущего. Если negative → чистим остальных и идём к Camunda.
            // Если positive (accept) → поднимаем следующего Waiting -> Pending и выходим.
            if (isSequential)
            {
                if (!Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var stage))
                    throw new InvalidOperationException($"Unknown process stage: {currentTask.BlockCode}");
                
                var parent = await unitOfWork.ProcessTaskRepository
                    .GetByIdAsync(cancellationToken, currentTask.ParentTaskId!.Value);

                var siblings = await unitOfWork.ProcessTaskRepository
                    .GetByFilterListAsync(cancellationToken, t => t.ParentTaskId == parent.Id);

                await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode,
                    cancellationToken);
                await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

                // текущего — убрать из кэша его Pending (важно для positive пути)
                await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);
                
                // Был ли отрицательный исход на текущем шаге?
                bool isNegative = stage switch
                {
                    ProcessStage.Rework   => command.Action == ProcessAction.Cancel,
                    ProcessStage.Signing  => command.Action == ProcessAction.Reject,
                    _ /* Approval и прочее */ => command.Condition is ProcessCondition.remake or ProcessCondition.reject
                };
                
                if (isNegative)
                {

                    // текущего — убрать из кэша его Pending
                    await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

                    var affectedAssignees = siblings
                        .Where(t => t.Status == "Waiting" && !string.IsNullOrEmpty(t.AssigneeCode))
                        .Select(t => t.AssigneeCode)
                        .Distinct()
                        .ToList();

                    foreach (var w in siblings.Where(t => t.Status == "Waiting"))
                    {
                        await processTaskService.FinalizeTaskAsync(w, cancellationToken);
                    }

                    // у тех, у кого были waiting — почистить
                    foreach (var a in affectedAssignees)
                        await processTaskService.RefreshUserTaskCacheAsync(a, cancellationToken);
                }
                else
                {
                    // accept: продолжаем цепочку
                    var next = siblings
                        .Where(t => t.Status == "Waiting")
                        .OrderBy(t => t.Order ?? int.MaxValue)
                        .ThenBy(t => t.Created)
                        .FirstOrDefault();

                    if (next != null)
                    {
                        await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
                        {
                            EntityId = next.Id,
                            Status = "Pending"
                        }, cancellationToken);

                        // у next теперь новая Pending — обновим его кэш
                        if (!string.IsNullOrEmpty(next.AssigneeCode))
                            await processTaskService.RefreshUserTaskCacheAsync(next.AssigneeCode, cancellationToken);

                        return new BaseResponseDto<SendProcessResponse>
                        {
                            Data = new SendProcessResponse { Success = true }
                        };
                    }

                    // сюда попадём, если очередников не осталось — пойдём сабмитить родителя как обычно
                }
            }


             var pendingSiblings = await unitOfWork.ProcessTaskRepository.CountAsync(
                     cancellationToken,
                     t => t.ParentTaskId == currentTask.ParentTaskId
                          && t.Id != currentTask.Id
                          && t.Status == "Pending");

            if (pendingSiblings > 0)
            {
                await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
                await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

                // текущего — убрать из кэша его Pending
                await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

                return new BaseResponseDto<SendProcessResponse>
                {
                    Data = new SendProcessResponse { Success = true }
                };
            }

            var parentTask =
                await unitOfWork.ProcessTaskRepository.GetByIdAsync(cancellationToken, currentTask.ParentTaskId.Value);

            if (parentTask.AssigneeCode == "system")
            {
                var claimedTasksResponse =
                    await camundaService.CamundaClaimedTasks(processData.ProcessInstanceId, parentTask.BlockCode)
                    ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);
                
                if (!Enum.TryParse<ProcessStage>(currentTask.BlockCode, out var stageForCamunda))
                    throw new InvalidOperationException($"Unknown process stage: {currentTask.BlockCode}");

                // >>> НОВОЕ: получаем правильные булевы переменные под конкретный этап
                var variables = await BuildCamundaVariablesAsync(
                    stageForCamunda,
                    command.Condition, // accept/remake/reject
                    command.Action, // <<< ДОБАВИЛИ
                    processData.Id,
                    parentTask.Id, // нужен для Approval-раунда
                    unitOfWork,
                    cancellationToken);

                var submitResponse = await camundaService.CamundaSubmitTask(
                    new CamundaSubmitTaskRequest
                    {
                        TaskId = claimedTasksResponse.TaskId,
                        Variables = variables
                    });

                if (!submitResponse.Success)
                {
                    throw new HandlerException($"Ошибка при отправке Msg:{submitResponse.Msg}", ErrorCodesEnum.Camunda);
                }

                await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
                await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
                await processTaskService.FinalizeTaskAsync(parentTask, cancellationToken);

                // иначе у части пользователей висит старая карточка
                if (!string.IsNullOrWhiteSpace(currentTask.AssigneeCode))
                    await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode!, cancellationToken);

                return new BaseResponseDto<SendProcessResponse>
                {
                    Data = new SendProcessResponse { Success = true }
                };
            }
            
            //Добавил 
            await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
            await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
            
            // финализировали current выше — теперь обновим его кэш один раз
            await processTaskService.RefreshUserTaskCacheAsync(currentTask.AssigneeCode, cancellationToken);

            // поднимем родителя
            await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
            {
                EntityId = parentTask.Id,
                Status = "Pending"
            }, cancellationToken);

            // у родителя появился Pending — обновим его кэш (если реальный исполнитель)
            if (!string.IsNullOrWhiteSpace(parentTask.AssigneeCode) &&
                !string.Equals(parentTask.AssigneeCode, "system", StringComparison.OrdinalIgnoreCase))
            {
                await processTaskService.RefreshUserTaskCacheAsync(parentTask.AssigneeCode, cancellationToken);
            }

            return new BaseResponseDto<SendProcessResponse>
            {
                Data = new SendProcessResponse { Success = true }
            };

        }
        
        private static string GetCamundaVarNameForStage(ProcessStage stage) => stage switch
        {
            ProcessStage.Approval => "agreement",
            ProcessStage.Signing => "sign",
            ProcessStage.Execution => "executed", // если в BPMN ждёшь другое имя — поменяй тут
            ProcessStage.ExecutionCheck => "checked", // при необходимости переименуй
            ProcessStage.Rework => "refinement", // <<< БЫЛО "agreement", теперь "refinement"
            _ => "agreement" // безопасный дефолт
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
            var varName = GetCamundaVarNameForStage(stage);

            switch (stage)
            {
                case ProcessStage.Rework:
                    // refinement: Submit => true, Cancel => false
                    return new Dictionary<string, object>
                    {
                        [varName] = (action == ProcessAction.Submit)
                    };

                case ProcessStage.Signing:
                    // sign: Submit => true, Reject => false
                    return new Dictionary<string, object>
                    {
                        [varName] = (action == ProcessAction.Submit)
                    };

                case ProcessStage.Approval:
                {
                    // agreement: accept => true, remake/reject => false
                    var isPositive = (condition == ProcessCondition.accept);

                    if (isPositive)
                    {
                        // «вето» в рамках того же родителя (раунда)
                        isPositive = await IsStagePositiveConsideringHistoryAsync(
                            ProcessStage.Approval, isPositive, processDataId, parentTaskId, unitOfWork, ct);
                    }

                    return new Dictionary<string, object> { [varName] = isPositive };
                }
                // 🔧 НОВОЕ: Execution — считаем позитивом и по Action, если Condition не задан
                        case ProcessStage.Execution:
                        {
                            bool isPositive = condition switch
                            {
                                ProcessCondition.accept => true,
                                ProcessCondition.remake or ProcessCondition.reject => false,
                                _ => action == ProcessAction.Submit
                            };
                            return new Dictionary<string, object> { [varName] = isPositive }; // varName == "executed"
                        }
                
                        // 🔧 НОВОЕ: ExecutionCheck — аналогично
                        case ProcessStage.ExecutionCheck:
                        {
                            bool isPositive = condition switch
                            {
                                ProcessCondition.accept => true,
                                ProcessCondition.remake or ProcessCondition.reject => false,
                                _ => action == ProcessAction.Submit
                            };
                            return new Dictionary<string, object> { [varName] = isPositive }; // varName == "checked"
                        }


                default:
                {
                    // дефолт: accept => true
                    var isPositive = (condition == ProcessCondition.accept);
                    return new Dictionary<string, object> { [varName] = isPositive };
                }
            }
        }

        /// <summary>
        /// true, если processData.approvalTypeCode == "Sequentially"
        /// </summary>
        private static bool ReadIsSequentialFromProcessData(Dictionary<string, object>? payload)
        {
            if (payload == null) return false;
            try
            {
                if (!payload.TryGetValue("processData", out var pdRaw) || pdRaw is null) return false;

                var json = JsonSerializer.Serialize(pdRaw);
                var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(json);

                if (pd != null && pd.TryGetValue("approvalTypeCode", out var typeRaw) && typeRaw is not null)
                {
                    var tJson = JsonSerializer.Serialize(typeRaw);
                    var type = JsonSerializer.Deserialize<string>(tJson);
                    return string.Equals(type, "Sequentially", StringComparison.OrdinalIgnoreCase);
                }
            }
            catch
            {
                /* считаем Parallel */
            }

            return false;
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
        
        public static T? ExtractFromPayload<T>(Dictionary<string, object>? payload, string key)
        {
            if (payload == null || !payload.TryGetValue(key, out var value))
                return default;

            var json = JsonSerializer.Serialize(value);
            return JsonSerializer.Deserialize<T>(json, new JsonSerializerOptions
            {
                PropertyNameCaseInsensitive = true
            });
        }
    }
}


