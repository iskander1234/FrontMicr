foreach (var emp in remaining)
{
    if (!insertedTabNumbers.Contains(emp.ManagerTabNumber!))
    {
        Console.WriteLine($"⚠️ Проблема: {emp.Name} не может быть вставлен. Менеджер с табельным номером {emp.ManagerTabNumber} ещё не добавлен.");
    }
}
