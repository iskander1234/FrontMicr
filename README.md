using BpmBaseApi.Domain.Entities.Event.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Commands.Process;
using BpmBaseApi.Shared.Enum;
using BpmBaseApi.Shared.Models.Camunda;
using BpmBaseApi.Shared.Models.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;
using System.Text.Encodings.Web;
using System.Text.Json;
using System.Text.Json.Nodes;
using BpmBaseApi.Shared.Dtos;
using static BpmBaseApi.Shared.Models.Process.CommonData;

namespace BpmBaseApi.Application.CommandHandlers.Process
{
    public class StartProcessCommandHandler(
        IUnitOfWork unitOfWork,
        IPayloadReaderService payloadReader,
        IProcessTaskService helperService,
        ICamundaService camundaService)
        : IRequestHandler<StartProcessCommand, BaseResponseDto<StartProcessResponse>>
    {
        public async Task<BaseResponseDto<StartProcessResponse>> Handle(
            StartProcessCommand command,
            CancellationToken cancellationToken)
        {
            var process = await unitOfWork.ProcessRepository
                              .GetByFilterAsync(cancellationToken, p => p.ProcessCode == command.ProcessCode)
                          ?? throw new HandlerException($"Процесс с кодом {command.ProcessCode} не найден",
                              ErrorCodesEnum.Business);

            var jsonOptions = new JsonSerializerOptions { Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping };
            var payloadJson = JsonSerializer.Serialize(command.Payload, jsonOptions);

            var regData = payloadReader.ReadSection<RegData>(command.Payload, "regData");
            var processDataDto = payloadReader.ReadSection<ProcessDataDto>(command.Payload, "processData");

            // NEW: читаем номер из payload.regData.regnum
            var regnumFromPayload = TryGetRegnumFromPayload(payloadJson);

            // === A) если regnum задан, пробуем использовать существующую заявку ===
            if (!string.IsNullOrWhiteSpace(regnumFromPayload))
            {
                var alreadyStarted = await unitOfWork.ProcessDataRepository.GetByFilterAsync(
                    cancellationToken,
                    p => p.ProcessCode == process.ProcessCode
                         && p.RegNumber == regnumFromPayload
                         && p.StatusCode == "Started"
                );
                if (alreadyStarted is not null)
                {
                    return new BaseResponseDto<StartProcessResponse>
                    {
                        Data = new StartProcessResponse
                            { ProcessGuid = alreadyStarted.Id, RegNumber = alreadyStarted.RegNumber }
                    };
                }

                var draft = await unitOfWork.ProcessDataRepository.GetByFilterAsync(
                    cancellationToken,
                    p => p.ProcessCode == process.ProcessCode
                         && p.RegNumber == regnumFromPayload
                         && p.StatusCode == "Draft"
                );

                if (draft is not null)
                {
                    // NEW: синхронизируем payload.regData.regnum с реальным номером черновика
                    payloadJson = SetRegnumInPayload(payloadJson, draft.RegNumber, jsonOptions);

                    await unitOfWork.ProcessDataRepository.RaiseEvent(new ProcessDataEditedEvent
                    {
                        EntityId = draft.Id,
                        StatusCode = "Started",
                        StatusName = "В работе",
                        PayloadJson = payloadJson,
                        InitiatorCode = regData?.UserCode,
                        InitiatorName = regData?.UserName,
                        Title = processDataDto?.DocumentTitle ?? draft.Title,
                        StartedAt = draft.Started
                        
                    }, cancellationToken);

                    await StartCamundaAndLinkAsync(draft.Id, cancellationToken);
                    await AddFilesUpsertAsync(draft.Id, payloadJson, cancellationToken);
                    await WriteHistoryStartIfAbsentAsync(draft.Id, draft.RegNumber, payloadJson, cancellationToken);

                    return new BaseResponseDto<StartProcessResponse>
                    {
                        Data = new StartProcessResponse { ProcessGuid = draft.Id, RegNumber = draft.RegNumber }
                    };
                }
            }

            // === B) черновика нет — создаём новую Started ===
            var requestNumber = await helperService.GenerateRequestNumberAsync(command.ProcessCode, cancellationToken);

            // NEW: вписываем номер в payload.regData.regnum
            payloadJson = SetRegnumInPayload(payloadJson, requestNumber, jsonOptions);

            var createdEvt = new ProcessDataCreatedEvent
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
                StartedAt = DateTime.Now
            };
            await unitOfWork.ProcessDataRepository.RaiseEvent(createdEvt, cancellationToken);

            await StartCamundaAndLinkAsync(createdEvt.EntityId, cancellationToken);
            await AddFilesUpsertAsync(createdEvt.EntityId, payloadJson, cancellationToken);
            await WriteHistoryStartIfAbsentAsync(createdEvt.EntityId, requestNumber, payloadJson, cancellationToken);

            return new BaseResponseDto<StartProcessResponse>
            {
                Data = new StartProcessResponse { ProcessGuid = createdEvt.EntityId, RegNumber = requestNumber }
            };

            // ---------- ЛОКАЛЬНЫЕ ФУНКЦИИ (NEW) ----------

            static string? TryGetRegnumFromPayload(string payload)
            {
                try
                {
                    var root = JsonNode.Parse(payload)!.AsObject();
                    return root["regData"]?["regnum"]?.GetValue<string>()?.Trim();
                }
                catch
                {
                    return null;
                }
            }

            static string SetRegnumInPayload(string payload, string regnum, JsonSerializerOptions opts)
            {
                var root = JsonNode.Parse(payload)!.AsObject();
                if (root["regData"] is not JsonObject rd)
                {
                    rd = new JsonObject();
                    root["regData"] = rd;
                }

                rd["regnum"] = regnum;
                return JsonSerializer.Serialize(root, opts);
            }

            async Task StartCamundaAndLinkAsync(Guid processDataId, CancellationToken ct)
            {
                var pi = await camundaService.CamundaStartProcess(new CamundaStartProcessRequest
                {
                    processCode = command.ProcessCode,
                    variables = new Dictionary<string, object> { { "processGuid", processDataId.ToString() } }
                });

                await unitOfWork.ProcessDataRepository.RaiseEvent(
                    new ProcessDataProcessInstanseIdChangedEvent
                    {
                        EntityId = processDataId,
                        ProcessInstanceId = pi
                    },
                    ct);
            }

            async Task AddFilesUpsertAsync(Guid processDataId, string payload, CancellationToken ct)
            {
                try
                {
                    var root = JsonNode.Parse(payload)!.AsObject();
                    var files = root["files"] as JsonArray;
                    if (files is null || files.Count == 0) return;

                    var existing = await unitOfWork.ProcessFileRepository.GetByFilterListAsync(
                        ct, f => f.ProcessDataId == processDataId);
                    var existingIds = existing.Where(x => x.FileId != Guid.Empty)
                        .Select(x => x.FileId)
                        .ToHashSet();

                    foreach (var node in files.OfType<JsonObject>())
                    {
                        var fileIdStr = node["fileId"]?.GetValue<string>();
                        var fileName = node["fileName"]?.GetValue<string>();
                        var fileType = node["fileType"]?.GetValue<string>();

                        if (string.IsNullOrWhiteSpace(fileName) || string.IsNullOrWhiteSpace(fileType))
                            throw new HandlerException(
                                "Каждый объект в \"files\" должен иметь непустые fileName и fileType",
                                ErrorCodesEnum.Business);
                        if (string.IsNullOrWhiteSpace(fileIdStr) || !Guid.TryParse(fileIdStr, out var fileId))
                            throw new HandlerException(
                                "Каждый объект в \"files\" должен иметь корректный GUID в поле fileId",
                                ErrorCodesEnum.Business);

                        if (existingIds.Contains(fileId)) continue;

                        await unitOfWork.ProcessFileRepository.RaiseEvent(new ProcessFileCreatedEvent
                        {
                            EntityId = Guid.NewGuid(),
                            ProcessDataId = processDataId,
                            FileId = fileId,
                            FileName = fileName,
                            FileType = fileType
                        }, ct);

                        existingIds.Add(fileId);
                    }
                }
                catch
                {
                    /* опционально залогировать */
                }
            }

            async Task WriteHistoryStartIfAbsentAsync(Guid processDataId, string regNumber, string payload,
                CancellationToken ct)
            {
                var hasStart = await unitOfWork.ProcessTaskHistoryRepository.CountAsync(
                    ct, h => h.ProcessDataId == processDataId && h.Action == ProcessAction.Start.ToString());
                if (hasStart > 0) return;

                await unitOfWork.ProcessTaskHistoryRepository.RaiseEvent(new ProcessTaskHistoryCreatedEvent
                {
                    ProcessDataId = processDataId,
                    TaskId = processDataId,
                    Action = ProcessAction.Start.ToString(),
                    BlockName = "Регистрационная форма",
                    Timestamp = DateTime.Now,
                    PayloadJson = payload,
                    Comment = "",
                    Description = "",
                    ProcessCode = command.ProcessCode,
                    ProcessName = process.ProcessName,
                    RegNumber = regNumber,
                    InitiatorCode = regData.UserCode,
                    InitiatorName = regData.UserName,
                    AssigneeCode = regData.UserCode,
                    AssigneeName = regData.UserName,
                    Title = processDataDto.DocumentTitle,
                    ActionName = "Зарегистрировано"
                }, ct);
            }
        }
    }
}



           // ★ 4) Сортировка по дате последней активности (max Timestamp из history) — по убыванию
            var lastActivityByPdId = history
                .GroupBy(h => h.ProcessDataId)
                .ToDictionary(g => g.Key, g => g.Max(x => x.Timestamp));

            var ordered = processes
                .OrderByDescending(p => lastActivityByPdId.TryGetValue(p.Id, out var ts) ? ts : DateTime.MinValue)
                .ToList();

            // 5) Маппим как было
            var mapped = mapper.Map<List<GetUserRelatedProcessesResponse>>(ordered);

            // ★ 6) Пагинация in-memory (порядок уже отсортирован по дате)
            var total = mapped.Count;
            var pageItems = mapped.ToPage(page, pageSize).ToList();

            // ★ 7) Возвращаем как и раньше BaseResponseDto<List<...>>,
            //     фактически — его наследник с полем pagination (не ломает фронт)
            return new PagedResponseDto<GetUserRelatedProcessesResponse>
            {
                Data = pageItems,
                Pagination = new PaginationMeta(total, page, pageSize)
            };
        }

