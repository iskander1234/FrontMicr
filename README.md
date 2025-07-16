/// <summary>
    /// Обновляет список сотрудников, сначала очищая таблицу,
    /// затем вставляя сотрудников с учётом зависимости от менеджеров.
    /// </summary>
    public async Task Update2(List<EmployeeEntity> employees)
    {
        try
        {
            // Удаление всех сотрудников
            var all = await _context.Employees.ToListAsync();
            _context.Employees.RemoveRange(all);
            await _context.SaveChangesAsync();

            // Вставка с учётом менеджеров
            await InsertWithManagerResolutionAsync(employees);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"❌ Ошибка при обновлении сотрудников: {ex.Message}");
            throw;
        }
    }

    /// <summary>
    /// Вставка с учётом внешнего ключа manager_tab_number.
    /// </summary>
    private async Task InsertWithManagerResolutionAsync(List<EmployeeEntity> employees)
    {
        var inserted = new List<EmployeeEntity>();
        var remaining = employees.ToList();

        while (remaining.Any())
        {
            var readyToInsert = remaining
                .Where(e => string.IsNullOrEmpty(e.ManagerTabNumber) ||
                            inserted.Any(i => i.TabNumber == e.ManagerTabNumber))
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


var employeeRepo = serviceProvider.GetRequiredService<EmployeeRepository>();
var employees = await LoadEmployeesAsync(); // или Deserialize, или GetFromLdap
await employeeRepo.InsertWithManagerResolutionAsync(employees);
