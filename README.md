public class SetProcessStageCommandHandler(
        IUnitOfWork unitOfWork,
        IProcessTaskService processTaskService
    ) : IRequestHandler<SetProcessStageCommand, BaseResponseDto<SetProcessStageResponse>>
    {
        public async Task<BaseResponseDto<SetProcessStageResponse>> Handle(SetProcessStageCommand request, CancellationToken cancellationToken)
        {
            var processData = await unitOfWork.ProcessDataRepository.GetByFilterAsync(
                cancellationToken, a => a.Id == request.ProcessGuid);

            if (processData is null)
                return new BaseResponseDto<SetProcessStageResponse> { Data = new SetProcessStageResponse { Status = "Ok" } };

            var processStageInfo = await unitOfWork.RefProcessStageRepository.GetByFilterAsync(
                cancellationToken, a => a.Code == request.Stage.ToString());

            await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataBlockChangedEvent
            {
                EntityId = processData.Id,
                BlockCode = processStageInfo.Code,
                BlockName = processStageInfo.Name,
            }, cancellationToken);

            var payloadDict = JsonSerializer.Deserialize<Dictionary<string, object>>(processData.PayloadJson ?? "{}");

            string key = request.Stage switch
            {
                ProcessStage.Approval      => "approvers",
                ProcessStage.Signing       => "signer",
                ProcessStage.Rework        => "initiator",
                ProcessStage.Execution     => "recipients",
                ProcessStage.ExecutionCheck=> "initiator",
                _ => throw new InvalidOperationException($"Unsupported process stage: {request.Stage}")
            };

            var recipients = ExtractFromPayloadFlexible<UserDataDto>(payloadDict, key) ?? new();

            // читаем режим из processData.approvalTypeCode
            bool isSequential = IsSequential(payloadDict);

            // создаём родительскую (system) задачу как раньше
            var systemTask = await processTaskService.CreateSystemTaskAsync(
                processStageInfo.Code, processStageInfo.Name, processData, cancellationToken);

            // --- СОГЛАСОВАНИЕ ---
            if (request.Stage == ProcessStage.Approval)
            {
                // порядок для всех режимов — берём из dto.Order; если пусто — автоинкремент
                var ordered = recipients
                    .Select((r, idx) => new { Item = r, Index = idx })
                    .OrderBy(x => x.Item.Order ?? int.MaxValue)
                    .ThenBy(x => x.Item.UserCode ?? string.Empty)
                    .ThenBy(x => x.Index)
                    .Select(x => x.Item)
                    .ToList();

                int seq = 1;
                bool isFirst = true;

                foreach (var r in ordered)
                {
                    var assignee = r.UserCode;
                    if (string.IsNullOrWhiteSpace(assignee)) continue;

                    await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
                    {
                        EntityId       = Guid.NewGuid(),
                        ProcessDataId  = processData.Id,
                        ParentTaskId   = systemTask.EntityId,

                        // обязательные поля (чтоб не словить NOT NULL)
                        ProcessCode    = processData.ProcessCode,
                        ProcessName    = processData.ProcessName ?? "",
                        RegNumber      = processData.RegNumber ?? "",
                        InitiatorCode  = processData.InitiatorCode ?? "",
                        InitiatorName  = processData.InitiatorName ?? "",
                        Title          = processData.Title ?? "",

                        AssigneeCode   = assignee.ToLowerInvariant(),
                        AssigneeName   = r.UserName ?? "",
                        BlockCode      = processStageInfo.Code,
                        BlockName      = processStageInfo.Name,
                        Status         = isSequential ? (isFirst ? "Pending" : "Waiting") : "Pending",
                        Order          = r.Order ?? seq
                    }, cancellationToken);

                    isFirst = false;
                    seq++;
                }

                return new BaseResponseDto<SetProcessStageResponse> { Data = new SetProcessStageResponse { Status = "Ok" } };
            }

            // для остальных стадий — как раньше (если нужно — оставляй существующую реализацию)
            await processTaskService.CreateTasksNewAsync(
                processStageInfo.Code, processStageInfo.Name, recipients, processData, systemTask.EntityId, cancellationToken);

            return new BaseResponseDto<SetProcessStageResponse> { Data = new SetProcessStageResponse { Status = "Ok" } };
        }

        private static bool IsSequential(Dictionary<string, object>? payload)
        {
            if (payload == null) return false;
            try
            {
                if (!payload.TryGetValue("processData", out var pdRaw) || pdRaw == null) return false;
                var pdJson = JsonSerializer.Serialize(pdRaw);
                var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(pdJson);
                var code = pd?["approvalTypeCode"]?.ToString();
                return string.Equals(code, "Sequentially", StringComparison.OrdinalIgnoreCase);
            }
            catch { return false; }
        }

        public static T? ExtractFromPayload<T>(Dictionary<string, object>? payload, string key)
        {
            if (payload == null || !payload.TryGetValue(key, out var value)) return default;
            var json = JsonSerializer.Serialize(value);
            return JsonSerializer.Deserialize<T>(json, new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
        }

        private List<T> ExtractFromPayloadFlexible<T>(Dictionary<string, object> payload, string key)
        {
            if (!payload.TryGetValue(key, out var rawValue) || rawValue == null) return new();
            var json = JsonSerializer.Serialize(rawValue);
            try { return JsonSerializer.Deserialize<List<T>>(json) ?? new(); }
            catch (JsonException)
            {
                try
                {
                    var single = JsonSerializer.Deserialize<T>(json);
                    return single != null ? new List<T> { single } : new();
                }
                catch (JsonException ex)
                {
                    throw new JsonException($"Не удалось десериализовать '{key}' как {typeof(T)} или List<{typeof(T)}>", ex);
                }
            }
        }
    }



     public class SendProcessCommandHandler(
        IMapper mapper,
        IUnitOfWork unitOfWork,
        IMemoryCache cache,
        ICamundaService camundaService,
        IProcessTaskService processTaskService)
        : IRequestHandler<SendProcessCommand, BaseResponseDto<SendProcessResponse>>
    {
        public async Task<BaseResponseDto<SendProcessResponse>> Handle(SendProcessCommand command, CancellationToken cancellationToken)
        {
            // берём задачу без фильтра статуса
            var currentTask = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                cancellationToken, p => p.Id == command.TaskId)
                ?? throw new HandlerException("Задача не найдена", ErrorCodesEnum.Business);

            var processData = await unitOfWork.ProcessDataRepository.GetByIdAsync(
                cancellationToken, currentTask.ProcessDataId)
                ?? throw new HandlerException("Заявка не найдена", ErrorCodesEnum.Business);

            var pdPayload = JsonSerializer.Deserialize<Dictionary<string, object>>(processData.PayloadJson ?? "{}");
            bool isSequential = IsSequential(pdPayload);

            // если последовательный режим, но задача не Pending — запрещаем
            if (isSequential && !string.Equals(currentTask.Status, "Pending", StringComparison.OrdinalIgnoreCase))
            {
                ProcessTaskEntity? activePending = null;

                if (currentTask.ParentTaskId != null)
                {
                    activePending = await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                        cancellationToken, t => t.ParentTaskId == currentTask.ParentTaskId && t.Status == "Pending");
                }

                activePending ??= await unitOfWork.ProcessTaskRepository.GetByFilterAsync(
                    cancellationToken, t => t.ProcessDataId == currentTask.ProcessDataId &&
                                            t.BlockCode == currentTask.BlockCode &&
                                            t.Status == "Pending");

                var who = activePending?.AssigneeCode ?? "не определён";
                throw new HandlerException(
                    $"Отправка запрещена: сначала должен отправить исполнитель со статусом Pending ({who}).",
                    ErrorCodesEnum.Business // у тебя это маппится в 400 BadRequest
                );
            }

            // старая защита на всякий случай
            if (!string.Equals(currentTask.Status, "Pending", StringComparison.OrdinalIgnoreCase))
                throw new HandlerException("Задача не найдена или уже обработана", ErrorCodesEnum.Business);

            var recipients = ExtractFromPayload<List<UserInfo>>(command.PayloadJson, "recipients") ?? new();
            var comment = ExtractFromPayload<string>(command.PayloadJson, "comment");

            // делегирование — без изменений
            if (command.Action == ProcessAction.Delegate && recipients.Any())
            {
                await processTaskService.HandleDelegationAsync(currentTask, processData, command, recipients, comment, cancellationToken);
                return processTaskService.SuccessResponse(currentTask.BlockCode, currentTask.BlockName);
            }

            if (isSequential)
            {
                // родитель и все дети
                var parent = await unitOfWork.ProcessTaskRepository.GetByIdAsync(
                    cancellationToken, currentTask.ParentTaskId!.Value);

                var siblings = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
                    cancellationToken, t => t.ParentTaskId == parent.Id);

                // лог и закрытие текущей
                await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
                await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

                // включаем следующего по Order (а если null — по Created)
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

                    return new BaseResponseDto<SendProcessResponse>
                    {
                        Data = new SendProcessResponse { Success = true }
                    };
                }
                // иначе — очередь завершена, дальше старая логика Camunda/родителя
            }

            // параллельный/старый сценарий — как было
            var pendingSiblings = await unitOfWork.ProcessTaskRepository.CountAsync(
                cancellationToken, t => t.ParentTaskId == currentTask.ParentTaskId && t.Id != currentTask.Id);

            if (pendingSiblings > 0)
            {
                await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
                await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

                return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };
            }

            var parentTask = await unitOfWork.ProcessTaskRepository.GetByIdAsync(
                cancellationToken, currentTask.ParentTaskId.Value);

            if (parentTask.AssigneeCode == "system")
            {
                var claimed = await camundaService.CamundaClaimedTasks(processData.ProcessInstanceId, parentTask.BlockCode)
                              ?? throw new HandlerException("Задача не найдена в Camunda", ErrorCodesEnum.Camunda);

                if (!Enum.TryParse<ProcessStage>(parentTask.BlockCode, out var stage))
                    throw new InvalidOperationException($"Unknown process stage: {parentTask.BlockCode}");

                var submit = await camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest
                {
                    TaskId = claimed.TaskId,
                    Variables = GetCamundaVariablesForStage(stage)
                });

                if (!submit.Success)
                    throw new HandlerException($"Ошибка при отправке Msg:{submit.Msg}", ErrorCodesEnum.Camunda);

                await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
                await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);
                await processTaskService.FinalizeTaskAsync(parentTask, cancellationToken);

                return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };
            }

            await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskStatusChangedEvent
            {
                EntityId = parentTask.Id,
                Status = "Pending"
            }, cancellationToken);

            return new BaseResponseDto<SendProcessResponse> { Data = new SendProcessResponse { Success = true } };
        }

        private static bool IsSequential(Dictionary<string, object>? payload)
        {
            if (payload == null) return false;
            try
            {
                if (!payload.TryGetValue("processData", out var pdRaw) || pdRaw == null) return false;
                var pdJson = JsonSerializer.Serialize(pdRaw);
                var pd = JsonSerializer.Deserialize<Dictionary<string, object>>(pdJson);
                var code = pd?["approvalTypeCode"]?.ToString();
                return string.Equals(code, "Sequentially", StringComparison.OrdinalIgnoreCase);
            }
            catch { return false; }
        }

        private Dictionary<string, object> GetCamundaVariablesForStage(ProcessStage stage) =>
            stage switch
            {
                ProcessStage.Approval => new() { { "agreement", true } },
                ProcessStage.Signing  => new() { { "sign", true } },
                _ => new()
            };

        public static T? ExtractFromPayload<T>(Dictionary<string, object>? payload, string key)
        {
            if (payload == null || !payload.TryGetValue(key, out var value)) return default;
            var json = JsonSerializer.Serialize(value);
            return JsonSerializer.Deserialize<T>(json, new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
        }
    }
