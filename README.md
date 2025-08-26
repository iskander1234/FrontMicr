// Если sequential — закрываем текущего. Если negative → чистим остальных и идём к Camunda.
// Если positive (accept) → поднимаем следующего Waiting -> Pending и выходим.
if (isSequential)
{
    var parent = await unitOfWork.ProcessTaskRepository
        .GetByIdAsync(cancellationToken, currentTask.ParentTaskId!.Value);

    var siblings = await unitOfWork.ProcessTaskRepository
        .GetByFilterListAsync(cancellationToken, t => t.ParentTaskId == parent.Id);

    await processTaskService.LogHistoryAsync(currentTask, command, currentTask.AssigneeCode, cancellationToken);
    await processTaskService.FinalizeTaskAsync(currentTask, cancellationToken);

    // Был ли отрицательный исход на текущем шаге?
    bool isNegative = command.Condition is ProcessCondition.remake or ProcessCondition.reject;

    if (isNegative)
    {
        // 1) Останавливаем цепочку: закрываем ВСЕ оставшиеся Waiting под тем же родителем
        foreach (var w in siblings.Where(t => t.Status == "Waiting"))
        {
            await processTaskService.FinalizeTaskAsync(w, cancellationToken);
        }

        // 2) Не поднимаем следующего. Дальше код пойдёт к parentTask (Camunda) и отправит false.
        // Просто выходим из этого if, НИЧЕГО не возвращаем здесь.
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

            // продолжаем ждать следующего исполнителя (в sequential мы живём шаг-за-шагом)
            return new BaseResponseDto<SendProcessResponse>
            {
                Data = new SendProcessResponse { Success = true }
            };
        }

        // сюда попадём, если очередников не осталось — пойдём сабмитить родителя как обычно
    }
}
