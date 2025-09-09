using System.Text.Json;
using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Implementations;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Enum;
using BpmBaseApi.Shared.Models.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using static BpmBaseApi.Shared.Models.Process.CommonData;
using static Microsoft.EntityFrameworkCore.DbLoggerCategory.Database;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    public class SetProcessStageCommandHandler(
        IUnitOfWork unitOfWork,
        IProcessTaskService processTaskService
        ) : IRequestHandler<SetProcessStageCommand, BaseResponseDto<SetProcessStageResponse>>
    {
        public async Task<BaseResponseDto<SetProcessStageResponse>> Handle(SetProcessStageCommand request, CancellationToken cancellationToken)
        {
            try
            {
                var processData = await unitOfWork
                .ProcessDataRepository
                .GetByFilterAsync(
                    cancellationToken,
                    a => a.Id == request.ProcessGuid
                );

            if (processData is null)
                return new BaseResponseDto<SetProcessStageResponse> { Data = new SetProcessStageResponse() { Status = "Ok" } };

            var processStageInfo = await unitOfWork.RefProcessStageRepository
                .GetByFilterAsync(
                    cancellationToken,
                    a => a.Code == request.Stage.ToString()
                );

            await unitOfWork
                .ProcessDataRepository
                .RaiseEvent(new ProcessDataBlockChangedEvent
                {
                    EntityId = processData.Id,
                    BlockCode = processStageInfo.Code,
                    BlockName = processStageInfo.Name,
                }, cancellationToken);
            
            if (request.Stage is ProcessStage.Completed or ProcessStage.Canceled)
            {
                return new BaseResponseDto<SetProcessStageResponse>
                {
                    Data = new SetProcessStageResponse { Status = "Ok" }
                };
            }


            var payloadDict = JsonSerializer.Deserialize<Dictionary<string, object>>(processData.PayloadJson);

            string key = request.Stage switch
            {
                ProcessStage.Approval => "approvers",
                ProcessStage.Signing => "signer",
                ProcessStage.Rework => "initiator",
                ProcessStage.Execution => "recipients",
                ProcessStage.ExecutionCheck => "initiator",
                _ => throw new InvalidOperationException($"Unsupported process stage: {request.Stage}")
            };

            var recipients = ExtractFromPayloadFlexible<UserDataDto>(payloadDict, key) ?? new();

            // 1) читаем тип согласования из processData.approvalTypeCode
            bool isSequential = ReadIsSequentialFromProcessData(payloadDict);

            // ---- VALIDATION: только для последовательного согласования ----
            if (request.Stage == ProcessStage.Approval && isSequential)
            {
                // 1) Проверяем, что у всех есть корректный Order (>0)
                var invalid = recipients
                    .Where(r => !r.Order.HasValue || r.Order!.Value <= 0)
                    .ToList();

                if (invalid.Any())
                {
                    var who = string.Join(", ",
                        invalid.Select(r => string.IsNullOrWhiteSpace(r.UserName) ? r.UserCode : r.UserName)
                            .Where(s => !string.IsNullOrWhiteSpace(s))
                            .Distinct()
                            .Take(5));
                    var tail = invalid.Count > 5 ? $" и ещё {invalid.Count - 5}" : "";
                    throw new HandlerException(
                        $"В последовательном режиме требуется указать порядок (Order > 0) для каждого участника. Не указан у: {who}{tail}.",
                        ErrorCodesEnum.Business);
                }

                // 2) Проверяем уникальность значений Order
                var duplicates = recipients
                    .GroupBy(r => r.Order!.Value)
                    .Where(g => g.Count() > 1)
                    .Select(g => g.Key)
                    .OrderBy(x => x)
                    .ToList();

                if (duplicates.Any())
                {
                    throw new HandlerException(
                        $"В последовательном режиме значения Order должны быть уникальны. Дублируются: {string.Join(", ", duplicates)}.",
                        ErrorCodesEnum.Business);
                }
            }
            
            // 2) родительская system‑задача — как раньше
            var systemTask = await processTaskService.CreateSystemTaskAsync(
                processStageInfo.Code, processStageInfo.Name, processData, cancellationToken);

            // 3) Если Approval и режим Sequentially — создаём детей Pending/Waiting и пишем Order
            if (request.Stage == ProcessStage.Approval && isSequential)
            {
                var ordered = recipients
                    .OrderBy(r => r.Order!.Value)  // важно: уже провалидировали, что Order есть
                    .ToList();

                bool isFirst = true;
                foreach (var r in ordered)
                {
                    var assignee = r.UserCode; // loginAD -> UserCode (как у тебя было)
                    if (string.IsNullOrWhiteSpace(assignee)) continue;

                    await unitOfWork.ProcessTaskRepository.RaiseEvent(new ProcessTaskCreatedEvent
                    {
                        EntityId       = Guid.NewGuid(),
                        ProcessDataId  = processData.Id,
                        ParentTaskId   = systemTask.EntityId,

                        ProcessCode    = processData.ProcessCode,
                        ProcessName    = processData.ProcessName ?? "",
                        RegNumber      = processData.RegNumber ?? "",
                        InitiatorCode  = processData.InitiatorCode ?? "",
                        InitiatorName  = processData.InitiatorName ?? "",
                        Title          = processData.Title ?? "",

                        AssigneeCode   = assignee.Trim().ToLowerInvariant(),
                        AssigneeName   = r.UserName ?? "",
                        BlockCode      = processStageInfo.Code,
                        BlockName      = processStageInfo.Name,
                        Status         = isFirst ? "Pending" : "Waiting",
                        Order          = r.Order!.Value // используем исходный Order (не перенумеровываем)
                    }, cancellationToken);

                    isFirst = false;
                }
            }
            else
            {
                // 4) Иначе — параллельно, как было. Метод внутри уже пишет Order
                await processTaskService.CreateTasksNewAsync(
                    processStageInfo.Code, processStageInfo.Name,
                    recipients, processData, systemTask.EntityId, cancellationToken);
            }

            return new BaseResponseDto<SetProcessStageResponse> { Data = new SetProcessStageResponse() { Status = "Ok" } };
            }
            // 1) наши бизнес-ошибки — просто пробрасываем, чтобы не потерять стек/сообщение
            catch (HandlerException)
            {
                throw;
            }
            // 2) отмену нельзя заворачивать
            catch (OperationCanceledException)
            {
                throw;
            }
            // 3) всё остальное — заворачиваем в HandlerException с inner (см. конструктор ниже)
            catch (Exception ex)
            {
                // логирование ex тут, если нужно
                throw new HandlerException($"Внутренняя ошибка при установке этапа процесса: {ex}", ErrorCodesEnum.Business);
            }

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

        private List<T> ExtractFromPayloadFlexible<T>(Dictionary<string, object> payload, string key)
        {
            if (!payload.TryGetValue(key, out var rawValue) || rawValue == null)
                return new();

            var json = JsonSerializer.Serialize(rawValue);

            try
            {
                // Пытаемся десериализовать как список
                return JsonSerializer.Deserialize<List<T>>(json) ?? new();
            }
            catch (JsonException)
            {
                try
                {
                    // Если не список — пробуем как одиночный объект и оборачиваем его в список
                    var singleItem = JsonSerializer.Deserialize<T>(json);
                    return singleItem != null ? new List<T> { singleItem } : new();
                }
                catch (JsonException ex)
                {
                    throw new JsonException($"Не удалось десериализовать поле '{key}' как {typeof(T)} или List<{typeof(T)}>", ex);
                }
            }
        }
        
        private static bool TryReadSequenceFlag(Dictionary<string, object>? payload)
        {
            if (payload == null) return false;
            try
            {
                if (!payload.TryGetValue("sysInfo", out var sysInfoRaw) || sysInfoRaw == null)
                    return false;

                var json = JsonSerializer.Serialize(sysInfoRaw);
                var sysInfo = JsonSerializer.Deserialize<Dictionary<string, object>>(json);

                if (sysInfo != null && sysInfo.TryGetValue("sequence", out var seqRaw))
                {
                    var seqJson = JsonSerializer.Serialize(seqRaw);
                    var seq = JsonSerializer.Deserialize<bool?>(seqJson);
                    return seq == true;
                }
            }
            catch { /* игнор, считаем false */ }

            return false;
        }

        /// <summary>
        /// true, если processData.approvalTypeCode == "Sequentially" (без учёта регистра)
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
            catch { /* считаем Parallel */ }

            return false;
        }
    }
}
