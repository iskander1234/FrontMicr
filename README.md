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
