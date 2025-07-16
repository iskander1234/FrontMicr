public async Task Update2(List<Employee> employees)
{
    var existing = await context.Employees.ToListAsync();

    // Пометить как уволенных
    foreach (var emp in existing)
    {
        emp.StatusCode = -1;
        emp.StatusDescription = "Уволен";
    }

    // Словарь уже вставленных сотрудников по табельному
    var inserted = existing.ToDictionary(x => x.TabNumber, x => x);
    var remaining = employees.ToList();

    // Основной цикл по вставке
    while (remaining.Any())
    {
        // Готовые к вставке: либо нет менеджера, либо менеджер уже вставлен
        var ready = remaining
            .Where(e => string.IsNullOrEmpty(e.ManagerTabNumber) || inserted.ContainsKey(e.ManagerTabNumber))
            .ToList();

        if (!ready.Any())
        {
            Console.WriteLine("❗ Не удалось вставить следующих сотрудников — менеджер не найден:");
            foreach (var emp in remaining)
                Console.WriteLine($"- {emp.Name} (Менеджер: {emp.ManagerTabNumber})");

            throw new Exception("Некоторые ManagerTabNumber отсутствуют в данных. Загрузка остановлена.");
        }

        foreach (var employee in ready)
        {
            var existingEmployee = existing.FirstOrDefault(e => e.TabNumber == employee.TabNumber);

            if (existingEmployee == null)
            {
                context.Employees.Add(employee);
                inserted[employee.TabNumber] = employee;
            }
            else
            {
                // обновляем данные
                existingEmployee.Name = employee.Name;
                existingEmployee.Position = employee.Position;
                existingEmployee.Login = employee.Login;
                existingEmployee.StatusCode = employee.StatusCode;
                existingEmployee.StatusDescription = employee.StatusDescription;
                existingEmployee.DepId = employee.DepId;
                existingEmployee.DepName = employee.DepName;
                existingEmployee.ParentDepId = employee.ParentDepId;
                existingEmployee.ParentDepName = employee.ParentDepName;
                existingEmployee.IsFilial = employee.IsFilial;
                existingEmployee.Mail = employee.Mail;
                existingEmployee.LocalPhone = employee.LocalPhone;
                existingEmployee.MobilePhone = employee.MobilePhone;
                existingEmployee.IsManager = employee.IsManager;
                existingEmployee.ManagerTabNumber = employee.ManagerTabNumber;
                existingEmployee.Disabled = employee.Disabled;
                existingEmployee.TabNumber = employee.TabNumber;

                inserted[employee.TabNumber] = existingEmployee;
            }
        }

        await context.SaveChangesAsync();

        // удаляем вставленные из очереди
        remaining = remaining.Except(ready).ToList();
    }

    Console.WriteLine($"✅ Всего обработано сотрудников: {inserted.Count}");
}
