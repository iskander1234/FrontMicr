public async Task Update2(List<EmployeeEntity> employees)
{
    try
    {
        await using var transaction = await _context.Database.BeginTransactionAsync();

        // 1. Удаляем старые записи
        var allEmployees = await _context.Employees.ToListAsync();
        _context.Employees.RemoveRange(allEmployees);
        await _context.SaveChangesAsync();

        // 2. Вставляем сотрудников без manager_id или с manager_id, которого пока нет
        var insertedTabNumbers = new HashSet<string>();

        // a. Вставляем сначала сотрудников без менеджеров
        var firstBatch = employees
            .Where(e => string.IsNullOrWhiteSpace(e.ManagerId))
            .ToList();

        _context.Employees.AddRange(firstBatch);
        await _context.SaveChangesAsync();
        insertedTabNumbers.UnionWith(firstBatch.Select(e => e.TabNumber));

        // b. Остальные вставляем по очереди — если их менеджер уже есть
        var remaining = employees
            .Where(e => !string.IsNullOrWhiteSpace(e.ManagerId) && !insertedTabNumbers.Contains(e.TabNumber))
            .ToList();

        int iteration = 0;
        const int maxIterations = 10;

        while (remaining.Any() && iteration < maxIterations)
        {
            var readyToInsert = remaining
                .Where(e => insertedTabNumbers.Contains(e.ManagerId))
                .ToList();

            if (!readyToInsert.Any())
                break;

            _context.Employees.AddRange(readyToInsert);
            await _context.SaveChangesAsync();
            insertedTabNumbers.UnionWith(readyToInsert.Select(e => e.TabNumber));
            remaining = remaining.Where(e => !insertedTabNumbers.Contains(e.TabNumber)).ToList();

            iteration++;
        }

        // 3. Логируем тех, кого не удалось вставить
        if (remaining.Any())
        {
            Console.WriteLine("❗ Не удалось вставить следующих сотрудников из-за отсутствующих менеджеров:");
            foreach (var emp in remaining)
            {
                Console.WriteLine($"- {emp.Name}, TabNumber: {emp.TabNumber}, ManagerId: {emp.ManagerId}");
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
