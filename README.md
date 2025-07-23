using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Internal;
using System;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;
using System.DirectoryServices.Protocols;
using System.Linq;
using System.Reflection;
using System.Runtime.Serialization;
using System.Text;
using System.Text.Json.Serialization;
using System.Threading.Tasks;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace DailyDataUpdateApp.Models;


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
    #region SetValueFromLdap

    public LDAPEmployee(in SearchResultEntry searchEntry)
    {
        var properties = typeof(LDAPEmployee).GetProperties();
        var entry = searchEntry;
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
            value = value
                .Split(',')[0]
                .Replace("CN=", "")
                .Trim();
        else if (prop.PropertyType == typeof(int) && int.TryParse(value, out var intValue))
        {
            if (jsonName == "UserAccountControl")
                IsDisabled = IsAccountDisabled(intValue);
            prop.SetValue(this, intValue);
        }
        else prop.SetValue(this, value);
        return value;
    }

    private void SetMemberOf(in string value)
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
    #endregion
    public void Configure(EntityTypeBuilder<LDAPEmployee> builder)
    {
        builder.ToTable("ldap_employees", "public",e => e.ExcludeFromMigrations());
        builder.HasKey(e => e.Id);
        builder.Property(e => e.Id).UseIdentityColumn();
        builder.HasIndex(e => e.Id);
        builder.Property(e => e.PositionName).IsRequired(false);
        builder.HasIndex(e => e.Login).IsUnique();
    }
}

using DailyDataUpdateApp.Interfaces.Shared.Repositories;
using DailyDataUpdateApp.Models;
using DailyDataUpdateApp.Shared.Contexes;
using EFCore.BulkExtensions;
using Microsoft.EntityFrameworkCore;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace DailyDataUpdateApp.Repositories
{
    internal class LDAPUsersRepository(BpmCoreUserContext userContext) : IListRepository<LDAPEmployee>
    {
        
        public async Task UpdateByKey(List<LDAPEmployee> updatedLdapEmployees)
        {
            await userContext.BulkInsertOrUpdateAsync(updatedLdapEmployees.ToList(), options =>
            {
                options.BatchSize = 1000;
                options.IncludeGraph = false;
                options.UpdateByProperties = ["Login"];
                options.SetOutputIdentity = true;
                options.UseTempDB = false;
                options.PropertiesToExclude =
                [
                    "Id", "GroupId", "DepartmentId",
                    //"DepartmentDetails",
                    "IsManager"
                ];
            });
        }
    }
}

using DailyDataUpdateApp.Interfaces.Complex.Services.Ldap;
using DailyDataUpdateApp.Models;
using Microsoft.Extensions.Configuration;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace DailyDataUpdateApp.Services.Ldap;
internal class UserLDAPService(IConfiguration configuration) : LDAPService(configuration), IUserLdapService
{
    public override string Filter => "UserPerson";

    public List<LDAPEmployee> GetLDAPEmployees()
    {
        var results = LdapPagedSearch();
        var employeeResults = results
            .Select(e => new LDAPEmployee(e))
            .ToList();
        return employeeResults;
    }
}

{
  "ConnectionStrings": {
    "BpmCore": "Host=localhost;Port=5432;Username=postgres;Password=postgres;Database=bpmbase5;SslMode=Disable;SearchPath=default"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "LDAPConfig": {
    "Server": "172.31.0.252",
    "Port": "389",
    "BindDN": "gnpfadm@enpf.kz",
    "Password": "EghfdktybtFLV$77#",
    "SearchBase": "dc=enpf,dc=kz",
    "Filters": {
      "UserPerson": {
        "Code": "(&(objectClass=user)(objectCategory=PERSON))",
        "Keys": [
          "givenName",
          "sAMAccountName",
          "department",
          "member",
          "memberof",
          "manager",
          "cn",
          "UserAccountControl&2",
          "UserAccountControl",
          "mail",
          "sn",
          "wWWWHomePage",
          "homePhone",
          "streetAddress",
          "postOfficeBox",
          "l",
          "st",
          "postalCode",
          "c",
          "company",
          "title",
          "telephoneNumber",
          "facsimileTelephoneNumber",
          "description",
          "ipPhone"
        ]
      },
      "Group": {
        "Code": "(objectCategory=group)",
        "Keys": [
          "name",
          "cn",
          "gidNumber",
          "description",
          "memberof",
          "primarygrouptoken",
          "primarygroupid",
          "samaccountname",
          "distinguishedname"
        ]
      },
      "User": "(objectClass=user)",
      "Person": "(objectCategory=PERSON)"
    }
  },
  "AllowedHosts": "*"
}

