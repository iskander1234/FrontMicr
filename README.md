using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using BpmBaseApi.Persistence.Interfaces;

namespace BpmBaseApi.Application.QueryHandlers.Process
{
    internal static class RoundScopeHelper
    {
        /// <summary>
        /// Собирает Id всех задач текущего «раунда», начиная с корневой (обычно system-задача),
        /// обходя всех детей по ParentTaskId (делегирование — сколько угодно уровней).
        /// </summary>
        public static async Task<HashSet<Guid>> CollectRoundScopeAsync(
            Guid rootId,
            IUnitOfWork uow,
            CancellationToken ct)
        {
            var scope = new HashSet<Guid> { rootId };
            var queue = new Queue<Guid>();
            queue.Enqueue(rootId);

            while (queue.Count > 0)
            {
                var pid = queue.Dequeue();

                var children = await uow.ProcessTaskRepository.GetByFilterListAsync(
                    ct,
                    t => t.ParentTaskId.HasValue && t.ParentTaskId.Value == pid
                );

                foreach (var ch in children)
                {
                    if (scope.Add(ch.Id))
                        queue.Enqueue(ch.Id);
                }
            }

            return scope;
        }
    }
}



using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using AutoMapper;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Enum;
using BpmBaseApi.Shared.Queries.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Application.QueryHandlers.Process
{
    public class GetExecutionProcessHistoryQueryHandler(
        IUnitOfWork unitOfWork,
        IMapper mapper,
        IProcessService processService)
        : IRequestHandler<GetExecutionProcessHistoryQuery, BaseResponseDto<List<GetExecutionProcessHistoryResponse>>>
    {
        public async Task<BaseResponseDto<List<GetExecutionProcessHistoryResponse>>> Handle(
            GetExecutionProcessHistoryQuery query,
            CancellationToken cancellationToken)
        {
            var processTask = await unitOfWork.ProcessTaskRepository
                .GetByFilterAsync(cancellationToken, t => t.Id == query.TaskId)
                ?? throw new HandlerException("Задача не найдена", ErrorCodesEnum.Business);

            // корень раунда: system-родитель, если есть; иначе сама задача
            var rootId = processTask.ParentTaskId ?? processTask.Id;

            // соберём scope текущего раунда
            var scopeIds = await RoundScopeHelper.CollectRoundScopeAsync(rootId, unitOfWork, cancellationToken);

            var execCode = ProcessStage.Execution.ToString();

            // История только по этому раунду: по ParentTaskId в scope ИЛИ по TaskId в scope
            var history = await unitOfWork.ProcessTaskHistoryRepository.GetByFilterListAsync(
                cancellationToken,
                h => h.ProcessDataId == processTask.ProcessDataId
                     && h.BlockCode == execCode
                     && (
                            (h.ParentTaskId.HasValue && scopeIds.Contains(h.ParentTaskId.Value))
                         || (h.TaskId != Guid.Empty && scopeIds.Contains(h.TaskId))
                        )
            );

            var historyItems = mapper.Map<List<GetProcessHistoryResponse>>(history.OrderBy(h => h.Timestamp));
            var tree = processService.BuildHistoryTree(historyItems);

            return new BaseResponseDto<List<GetExecutionProcessHistoryResponse>>
            {
                Data = mapper.Map<List<GetExecutionProcessHistoryResponse>>(tree)
            };
        }
    }
}



using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using AutoMapper;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Enum;
using BpmBaseApi.Shared.Queries.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Application.QueryHandlers.Process
{
    public class GetExecutionProcessDataHistoryQueryHandler(
        IUnitOfWork unitOfWork,
        IMapper mapper,
        IProcessService processService)
        : IRequestHandler<GetExecutionProcessDataHistoryQuery, BaseResponseDto<List<GetExecutionProcessHistoryResponse>>>
    {
        public async Task<BaseResponseDto<List<GetExecutionProcessHistoryResponse>>> Handle(
            GetExecutionProcessDataHistoryQuery query,
            CancellationToken cancellationToken)
        {
            var execCode = ProcessStage.Execution.ToString();

            // берём последний «корень раунда» Execution (system-задача)
            var possibleRoots = await unitOfWork.ProcessTaskRepository.GetByFilterListAsync(
                cancellationToken,
                t => t.ProcessDataId == query.ProcessDataId
                     && t.BlockCode == execCode
                     && t.AssigneeCode == "system"
            );

            var root = possibleRoots
                .OrderByDescending(t => t.Created)
                .FirstOrDefault();

            if (root is null)
            {
                // fallback: как было раньше — вся история Execution по заявке
                var historyAll = await unitOfWork.ProcessTaskHistoryRepository.GetByFilterListAsync(
                    cancellationToken,
                    h => h.ProcessDataId == query.ProcessDataId && h.BlockCode == execCode
                );

                var historyItemsAll = mapper.Map<List<GetProcessHistoryResponse>>(historyAll.OrderBy(h => h.Timestamp));
                var treeAll = processService.BuildHistoryTree(historyItemsAll);

                return new BaseResponseDto<List<GetExecutionProcessHistoryResponse>>
                {
                    Data = mapper.Map<List<GetExecutionProcessHistoryResponse>>(treeAll)
                };
            }

            // scope текущего раунда
            var scopeIds = await RoundScopeHelper.CollectRoundScopeAsync(root.Id, unitOfWork, cancellationToken);

            var history = await unitOfWork.ProcessTaskHistoryRepository.GetByFilterListAsync(
                cancellationToken,
                h => h.ProcessDataId == query.ProcessDataId
                     && h.BlockCode == execCode
                     && (
                            (h.ParentTaskId.HasValue && scopeIds.Contains(h.ParentTaskId.Value))
                         || (h.TaskId != Guid.Empty && scopeIds.Contains(h.TaskId))
                        )
            );

            var historyItems = mapper.Map<List<GetProcessHistoryResponse>>(history.OrderBy(h => h.Timestamp));
            var tree = processService.BuildHistoryTree(historyItems);

            return new BaseResponseDto<List<GetExecutionProcessHistoryResponse>>
            {
                Data = mapper.Map<List<GetExecutionProcessHistoryResponse>>(tree)
            };
        }
    }
}
