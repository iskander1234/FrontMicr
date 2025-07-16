// DinDin/Models/Ldap/LDAPEmployee.cs
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using System.ComponentModel.DataAnnotations.Schema;
using System.Text.Json.Serialization;
using System.DirectoryServices.Protocols;
using System.Reflection;

namespace DinDin.Models.Ldap;

[Serializable]
public record LDAPEmployee : IEntityTypeConfiguration<LDAPEmployee>
{
    [Column(name: "id")]
    public int Id { get; set; }

    [Column(name: "user_status_code")]
    [JsonPropertyName("UserAccountControl")]
    public int UserStatusCode { get; set; }

    [Column(name: "is_disabled")]
    public bool IsDisabled { get; set; }

    [Column(name: "given_name", TypeName = "varchar")]
    [JsonPropertyName("givenName")]
    public string? GivenName { get; set; } = null!;

    [Column(name: "name", TypeName = "varchar")]
    [JsonPropertyName("cn")]
    public string Name { get; set; } = null!;

    [Column(name: "position_name", TypeName = "varchar")]
    [JsonPropertyName("description")]
    public string? PositionName { get; set; } = null!;

    [Column(name: "title", TypeName = "varchar")]
    [JsonPropertyName("title")]
    public string? Title { get; set; } = null!;

    [Column(name: "department", TypeName = "varchar")]
    [JsonPropertyName("department")]
    public string? Department { get; set; } = null!;

    [Column(name: "login", TypeName = "varchar")]
    [JsonPropertyName("sAMAccountName")]
    public string Login { get; set; } = null!;

    [Column(name: "local_phone", TypeName = "varchar")]
    [JsonPropertyName("ipPhone")]
    public string? LocalPhone { get; set; } = null!;

    [Column(name: "mobile_number", TypeName = "varchar")]
    [JsonPropertyName("telephoneNumber")]
    public string? MobileNumber { get; set; } = null!;

    [Column(name: "city", TypeName = "varchar")]
    [JsonPropertyName("l")]
    public string? City { get; set; } = null!;

    [Column(name: "location_address", TypeName = "varchar")]
    [JsonPropertyName("streetAddress")]
    public string? LocationAddress { get; set; } = null!;

    [Column(name: "postal_code", TypeName = "varchar")]
    [JsonPropertyName("postalCode")]
    public string? PostalCode { get; set; } = null!;

    [Column(name: "company", TypeName = "varchar")]
    [JsonPropertyName("company")]
    public string? Company { get; set; } = null!;

    [Column(name: "manager", TypeName = "varchar")]
    [JsonPropertyName("manager")]
    public string? Manager { get; set; } = null!;

    [Column(name: "email", TypeName = "varchar")]
    [JsonPropertyName("mail")]
    public string? Email { get; set; } = null!;

    [Column(name: "member_of")]
    [JsonPropertyName("memberof")]
    public List<string>? MemberOf { get; set; } = null!;

    [Column(name: "member_of_string", TypeName = "varchar")]
    public string? MemberOfString { get; set; } = null!;

    [Column(name: "is_manager")]
    public bool IsManager { get; set; } = false;

    [Column(name: "group_id")]
    public int? GroupId { get; set; } = null!;

    [Column(name: "department_id")]
    public int? DepartmentId { get; set; } = null!;

    [Column(name: "department_details", TypeName = "jsonb")]
    public List<DepartmentDetail>? DepartmentDetails { get; set; } = null!;

    public LDAPEmployee() { }

    public LDAPEmployee(in SearchResultEntry entry)
    {
        var properties = typeof(LDAPEmployee).GetProperties();

        foreach (var property in properties)
        {
            var jsonName = property.GetCustomAttribute<JsonPropertyNameAttribute>()?.Name;

            if (jsonName == null || !entry.Attributes.Contains(jsonName))
                continue;

            var value = entry.Attributes[jsonName][0]?.ToString();
            if (string.IsNullOrEmpty(value))
                return;

            SetByJsonName(property, jsonName, value);
        }
    }

    private string? SetByJsonName(in PropertyInfo prop, string? jsonName, string value)
    {
        if (jsonName == "memberof")
            SetMemberOf(value);
        else if (jsonName == "manager" && value.Contains(','))
            value = value.Split(',')[0].Replace("CN=", "").Trim();
        else if (prop.PropertyType == typeof(int) && int.TryParse(value, out var intValue))
        {
            if (jsonName == "UserAccountControl")
                IsDisabled = IsAccountDisabled(intValue);
            prop.SetValue(this, intValue);
        }
        else prop.SetValue(this, value);

        return value;
    }

    private void SetMemberOf(string value)
    {
        MemberOf ??= [];
        MemberOfString = value.Replace(",DC=enpf,DC=kz", "");
        MemberOf = MemberOfString!
            .Replace(",OU=Группы", "")
            .Replace(",CN=Users", "")
            .Replace("OU=", "")
            .Replace("CN=", "")
            .Split(",")
            .Distinct()
            .ToList();
    }

    static bool IsAccountDisabled(int userAccountControlValue)
    {
        const int ACCOUNTDISABLE = 0x0002;
        return (userAccountControlValue & ACCOUNTDISABLE) == ACCOUNTDISABLE;
    }

    public void Configure(EntityTypeBuilder<LDAPEmployee> builder)
    {
        builder.ToTable("ldap_employees", "public", e => e.ExcludeFromMigrations());
        builder.HasKey(e => e.Id);
        builder.Property(e => e.Id).UseIdentityColumn();
        builder.HasIndex(e => e.Id);
        builder.Property(e => e.PositionName).IsRequired(false);
        builder.HasIndex(e => e.Login).IsUnique();
    }
}

----------

using DinDin.Models.Ldap;
using Microsoft.EntityFrameworkCore;
using EFCore.BulkExtensions;

namespace DinDin.Repositories.Ldap;

public class LDAPUsersRepository
{
    private readonly YourDbContext _context; // замени на свой контекст (например, BpmCoreUserContext)

    public LDAPUsersRepository(YourDbContext context)
    {
        _context = context;
    }

    public async Task UpdateByKey(List<LDAPEmployee> updatedLdapEmployees)
    {
        await _context.BulkInsertOrUpdateAsync(updatedLdapEmployees, options =>
        {
            options.BatchSize = 1000;
            options.IncludeGraph = false;
            options.UpdateByProperties = ["Login"];
            options.SetOutputIdentity = true;
            options.UseTempDB = false;
            options.PropertiesToExclude =
            [
                "Id", "GroupId", "DepartmentId",
                "IsManager"
            ];
        });
    }
}


--------------

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ApplyConfiguration(new LDAPEmployee());
}


-----------

private static async Task SyncLdapAsync(IServiceProvider serviceProvider)
{
    var configuration = serviceProvider.GetRequiredService<IConfiguration>();
    var context = serviceProvider.GetRequiredService<BpmCoreUserContext>();

    var ldapService = new UserLDAPService(configuration);
    var employees = ldapService.GetLDAPEmployees();

    var repository = new LDAPUsersRepository(context);
    await repository.UpdateByKey(employees);

    Console.WriteLine($"✅ Синхронизация LDAP завершена. Добавлено или обновлено: {employees.Count} записей");
}


--------

private static async Task StartData(ServiceProvider serviceProvider)
{
    try
    {
        var repo = serviceProvider.GetRequiredService<EmployeeRepository>();
        await repo.Update2(await repo.GetAllEmployeesAsync());
        await repo.InsertWithManagerResolutionAsync();

        // 👇 добавь этот вызов
        await SyncLdapAsync(serviceProvider);
    }
    catch (Exception ex)
    {
        Console.WriteLine($"❌ Ошибка при запуске данных: {ex.Message}");
    }
}
