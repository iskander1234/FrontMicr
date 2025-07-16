public async Task Update2(List<Employee> employees)
{
    using var transaction = await context.Database.BeginTransactionAsync();

    try
    {
        var existingEmployees = await context.Employees.ToListAsync();

        // 1. Помечаем всех как уволенных
        foreach (var e in existingEmployees)
        {
            e.StatusCode = -1;
            e.StatusDescription = "Уволен";
        }

        await context.SaveChangesAsync();

        // 2. Хранилище уже добавленных табельных номеров
        var insertedTabNumbers = new HashSet<string>(existingEmployees.Select(e => e.TabNumber));

        // 3. Группируем новых по наличию менеджера
        var noManager = employees
            .Where(e => string.IsNullOrEmpty(e.ManagerTabNumber))
            .ToList();

        context.Employees.AddRange(noManager);
        await context.SaveChangesAsync();
        foreach (var e in noManager)
            insertedTabNumbers.Add(e.TabNumber);

        var remaining = employees.Except(noManager).ToList();
        var added = true;

        while (remaining.Any() && added)
        {
            added = false;

            var nextBatch = remaining
                .Where(e => insertedTabNumbers.Contains(e.ManagerTabNumber))
                .ToList();

            if (nextBatch.Any())
            {
                context.Employees.AddRange(nextBatch);
                await context.SaveChangesAsync();

                foreach (var e in nextBatch)
                    insertedTabNumbers.Add(e.TabNumber);

                remaining = remaining.Except(nextBatch).ToList();
                added = true;
            }
        }

        if (remaining.Any())
        {
            var unresolved = string.Join(", ", remaining.Select(e => $"{e.Name} → {e.ManagerTabNumber}"));
            throw new Exception($"Ошибка внешнего ключа: не найдены менеджеры для: {unresolved}");
        }

        await transaction.CommitAsync();
    }
    catch (Exception ex)
    {
        await transaction.RollbackAsync();
        Console.WriteLine("❌ Ошибка при вставке сотрудников: " + ex.Message);
        throw;
    }
}
