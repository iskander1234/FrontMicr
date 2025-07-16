public async Task Update2(List<Employee> employees)
{
    try
    {
        await using var transaction = await context.Database.BeginTransactionAsync();

        // 1. Удаляем всех старых сотрудников
        var existing = await context.Employees.ToListAsync();
        context.Employees.RemoveRange(existing);
        await context.SaveChangesAsync();

        // 2. Вставляем сотрудников по волнам
        var insertedTabNumbers = new HashSet<string>();

        // 2.1. Первая волна — без менеджера
        var firstBatch = employees
            .Where(e => string.IsNullOrWhiteSpace(e.ManagerTabNumber))
            .ToList();

        context.Employees.AddRange(firstBatch);
        await context.SaveChangesAsync();
        insertedTabNumbers.UnionWith(firstBatch.Select(e => e.TabNumber));

        // 2.2. Оставшиеся, у кого есть менеджер
        var remaining = employees
            .Where(e => !string.IsNullOrWhiteSpace(e.ManagerTabNumber) && !insertedTabNumbers.Contains(e.TabNumber))
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

        // 3. Лог не вставленных
        if (remaining.Any())
        {
            Console.WriteLine("❗ Не удалось вставить сотрудников из-за отсутствующих менеджеров:");
            foreach (var emp in remaining)
            {
                Console.WriteLine($"- {emp.Name}, Таб. номер: {emp.TabNumber}, Менеджер: {emp.ManagerTabNumber}");
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
