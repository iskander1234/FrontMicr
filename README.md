public async Task Update2(List<Employee> employees)
{
    try
    {
        await using var context = new BpmcoreContext();

        // Логика: найти всех менеджеров из входного списка
        var allTabNumbers = new HashSet<string>(employees.Select(e => e.TabNumber));
        var allManagers = employees
            .Where(e => !string.IsNullOrWhiteSpace(e.ManagerTabNumber))
            .Select(e => e.ManagerTabNumber!)
            .Distinct()
            .ToList();

        // Найти тех менеджеров, которых нет ни в новой партии, ни в базе
        var notFoundManagers = allManagers
            .Where(managerTab => !allTabNumbers.Contains(managerTab) &&
                                 !context.Employees.Any(e => e.TabNumber == managerTab))
            .ToList();

        if (notFoundManagers.Any())
        {
            Console.WriteLine("❌ Найдены сотрудники с некорректными ManagerTabNumber:");
            foreach (var m in notFoundManagers)
            {
                var brokenEmployees = employees.Where(e => e.ManagerTabNumber == m);
                foreach (var emp in brokenEmployees)
                {
                    Console.WriteLine($"Сотрудник: {emp.Name} | TabNumber: {emp.TabNumber} | ManagerTabNumber: {emp.ManagerTabNumber}");
                }
            }

            throw new Exception("Невозможно сохранить: есть ссылки на несуществующих менеджеров.");
        }

        // Удаляем старые записи (опционально, если нужно обновление)
        context.Employees.RemoveRange(context.Employees);
        await context.SaveChangesAsync();

        // Добавляем новые
        await context.Employees.AddRangeAsync(employees);
        await context.SaveChangesAsync();

        Console.WriteLine("✅ Сотрудники успешно обновлены.");
    }
    catch (Exception ex)
    {
        Console.WriteLine($"❌ Ошибка при вставке сотрудников: {ex.Message}");
        throw;
    }
}
