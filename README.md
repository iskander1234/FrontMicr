public async Task InsertWithManagerResolutionAsync(List<EmployeeEntity> employees)
{
    var inserted = new List<EmployeeEntity>();
    var remaining = employees.ToList();

    while (remaining.Any())
    {
        var readyToInsert = remaining
            .Where(e => string.IsNullOrEmpty(e.ManagerTabNumber) || inserted.Any(i => i.TabNumber == e.ManagerTabNumber))
            .ToList();

        if (!readyToInsert.Any())
        {
            Console.WriteLine("❗ Остановлено: остались сотрудники с несуществующими ManagerTabNumber:");
            foreach (var emp in remaining)
                Console.WriteLine($"- {emp.Name} (TabNumber: {emp.TabNumber}) → Manager: {emp.ManagerTabNumber}");

            throw new Exception("Некоторые ManagerTabNumber не найдены среди загружаемых сотрудников. Загрузка остановлена.");
        }

        _context.Employees.AddRange(readyToInsert);
        await _context.SaveChangesAsync();

        inserted.AddRange(readyToInsert);
        remaining = remaining.Except(readyToInsert).ToList();
    }

    Console.WriteLine($"✅ Всего вставлено сотрудников: {inserted.Count}");
}
