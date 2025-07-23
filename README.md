public async Task ReplaceAllLdapEmployees(List<LDAPEmployee> employees)
{
    // Очищаем таблицу
    await _context.Database.ExecuteSqlRawAsync("TRUNCATE TABLE \"ldap_employees\" RESTART IDENTITY");

    // Вставляем по батчам (например, по 1000)
    var bulkConfig = new BulkConfig
    {
        BatchSize = 1000,         // Можно увеличить до 2000–5000
        UseTempDB = true          // Улучшает производительность
    };

    await _context.BulkInsertAsync(employees, bulkConfig);
}
