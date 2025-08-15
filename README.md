var shortName = GetShortName(emp.Name);

ShortName = shortName, // <---- добавили

// ========================== Общий приватный метод ==========================
private static string GetShortName(string? fullName)
{
    if (string.IsNullOrWhiteSpace(fullName))
        return "";

    var parts = fullName.Split(' ', StringSplitOptions.RemoveEmptyEntries);

    if (parts.Length == 0)
        return "";

    if (parts.Length == 1)
        return parts[0];

    var sb = new System.Text.StringBuilder();
    sb.Append(parts[0]); // Фамилия
    sb.Append(' ');
    sb.Append(char.ToUpper(parts[1][0])); // первая буква имени
    sb.Append('.');
    if (parts.Length >= 3 && !string.IsNullOrWhiteSpace(parts[2]))
    {
        sb.Append(char.ToUpper(parts[2][0])); // первая буква отчества
        sb.Append('.');
    }
    return sb.ToString();
}
