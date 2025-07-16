public async Task Update2(List<Employee> employees)
{
    try
    {
        await using var context = new BpmcoreContext();

        // Удаляем всех старых сотрудников (если нужно)
        context.Employees.RemoveRange(context.Employees);
        await context.SaveChangesAsync();

        // 1. Вставляем сотрудников без менеджера
        var withoutManagers = employees.Where(e => string.IsNullOrWhiteSpace(e.ManagerTabNumber)).ToList();
        await context.Employees.AddRangeAsync(withoutManagers);
        await context.SaveChangesAsync();

        // 2. Вставляем остальных по очереди, проверяя наличие менеджера
        var remaining = employees.Where(e => !string.IsNullOrWhiteSpace(e.ManagerTabNumber)).ToList();

        var insertedTabNumbers = new HashSet<string>(withoutManagers.Select(e => e.TabNumber));

        bool added;
        do
        {
            added = false;

            var insertable = remaining
                .Where(e => insertedTabNumbers.Contains(e.ManagerTabNumber!))
                .ToList();

            if (insertable.Any())
            {
                await context.Employees.AddRangeAsync(insertable);
                await context.SaveChangesAsync();

                foreach (var e in insertable)
                    insertedTabNumbers.Add(e.TabNumber);

                remaining = remaining.Except(insertable).ToList();
                added = true;
            }
        }
        while (added);

        // 3. Проверка: остались ли сотрудники с несуществующими менеджерами
        if (remaining.Any())
        {
            Console.WriteLine("❌ Не удалось вставить следующих сотрудников из-за отсутствия менеджеров:");
            foreach (var e in remaining)
                Console.WriteLine($"- {e.Name} | Таб. № {e.TabNumber} | Менеджер: {e.ManagerTabNumber}");

            throw new Exception("Не все сотрудники были добавлены из-за отсутствия менеджеров в базе.");
        }

        Console.WriteLine("✅ Все сотрудники успешно добавлены.");
    }
    catch (Exception ex)
    {
        Console.WriteLine($"❌ Ошибка при вставке сотрудников: {ex.Message}");
        throw;
    }
}
