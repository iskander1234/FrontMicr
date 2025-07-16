public async Task Update2(List<Employee> employees)
{
    try
    {
        await using var transaction = await context.Database.BeginTransactionAsync();

        // 1. Удаляем старые записи
        var allEmployees = await context.Employees.ToListAsync();
        context.Employees.RemoveRange(allEmployees);
        await context.SaveChangesAsync();

        // 2. Вставляем сначала тех, у кого нет менеджера
        var insertedTabNumbers = new HashSet<string>();

        var firstBatch = employees
            .Where(e => string.IsNullOrWhiteSpace(e.ManagerTabNumber))
            .ToList();

        context.Employees.AddRange(firstBatch);
        await context.SaveChangesAsync();
        insertedTabNumbers.UnionWith(firstBatch.Select(e => e.TabNumber));

        // 3. Остальные — только если их менеджер уже вставлен
        var remaining = employees
            .Where(e => !string.IsNullOrWhiteSpace(e.ManagerTabNumber) &&
                        !insertedTabNumbers.Contains(e.TabNumber))
            .ToList();

        int iteration = 0;
        const int maxIterations = 10;

        while (remaining.Any() && iteration < maxIterations)
        {
            var readyToInsert = remaining
                .Where(e => insertedTabNumbers.Contains(e.ManagerTabNumber))
                .ToList();

            if (!readyToInsert.Any())
                break;

            context.Employees.AddRange(readyToInsert);
            await context.SaveChangesAsync();

            insertedTabNumbers.UnionWith(readyToInsert.Select(e => e.TabNumber));
            remaining = remaining
                .Where(e => !insertedTabNumbers.Contains(e.TabNumber))
                .ToList();

            iteration++;
        }

        // 4. Лог
        if (remaining.Any())
        {
            Console.WriteLine("❗ Не удалось вставить следующих сотрудников из-за отсутствующих менеджеров:");
            foreach (var emp in remaining)
            {
                Console.WriteLine($"- {emp.Name}, TabNumber: {emp.TabNumber}, ManagerTabNumber: {emp.ManagerTabNumber}");
            }
        }

        await transaction.CommitAsync();
        Console.WriteLine("✅ Сотрудники успешно обновлены.");
    }
    catch (Exception ex)
    {
        Console.WriteLine($"❌ Ошибка при вставке сотрудников: {ex.Message}");
        throw;
    }
}
