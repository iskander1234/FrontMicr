using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;
using System.DirectoryServices;
using System.DirectoryServices.Protocols;
using System.Reflection;
using System.Text.Json;
using System.Text.Json.Serialization;
using DinDin.Extensions;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace DinDin.Models;

 [Table("ldap_employees")]
 public class LDAPEmployee
 {
     [Key]
     [Column("id")]
     public int Id { get; set; }

     [Column("user_status_code")]
     public int UserStatusCode { get; set; }

     [Column("is_disabled")]
     public bool IsDisabled { get; set; }

     [Column("given_name")]
     public string? GivenName { get; set; }

     [Column("name")]
     public string Name { get; set; } = "";

     [Column("position_name")]
     public string? PositionName { get; set; }

     [Column("title")]
     public string? Title { get; set; }

     [Column("department")]
     public string? Department { get; set; }

     [Column("login")]
     public string Login { get; set; } = "";

     [Column("local_phone")]
     public string? LocalPhone { get; set; }

     [Column("mobile_number")]
     public string? MobileNumber { get; set; }

     [Column("city")]
     public string? City { get; set; }

     [Column("location_address")]
     public string? LocationAddress { get; set; }

     [Column("postal_code")]
     public string? PostalCode { get; set; }

     [Column("company")]
     public string? Company { get; set; }

     [Column("manager")]
     public string? Manager { get; set; }

     [Column("email")]
     public string? Email { get; set; }

     [Column("member_of")]
     public string[]? MemberOf { get; set; }

     [Column("member_of_string")]
     public string? MemberOfString { get; set; }

     [Column("department_id")]
     public int? DepartmentId { get; set; }

     [Column("department_details")]
     public JsonDocument? DepartmentDetails { get; set; }

     [Column("group_id")]
     public int? GroupId { get; set; }

     [Column("is_manager")]
     public bool IsManager { get; set; }

     public LDAPEmployee() {}

     public LDAPEmployee(SearchResultEntry entry)
     {
         UserStatusCode = entry.GetInt("userAccountControl");
         IsDisabled = (UserStatusCode & 2) != 0;
         GivenName = entry.GetString("givenName");
         Name = entry.GetString("cn") ?? "";
         PositionName = entry.GetString("position");
         Title = entry.GetString("title");
         Department = entry.GetString("department");
         Login = entry.GetString("sAMAccountName") ?? "";
         LocalPhone = entry.GetString("telephoneNumber");
         MobileNumber = entry.GetString("mobile");
         City = entry.GetString("l");
         LocationAddress = entry.GetString("streetAddress");
         PostalCode = entry.GetString("postalCode");
         Company = entry.GetString("company");
         Manager = entry.GetString("manager");
         Email = entry.GetString("mail");
         MemberOf = entry.GetStringArray("memberOf");
         MemberOfString = MemberOf == null ? null : string.Join(";", MemberOf);
         IsManager = false;
     }
}



using DinDin.Models;
using EFCore.BulkExtensions;
using Microsoft.EntityFrameworkCore;

namespace DinDin.Repositories
{
    public class LDAPUsersRepository
    {
        private readonly BpmcoreContext _ctx;
        public LDAPUsersRepository(BpmcoreContext ctx) => _ctx = ctx;

        /// <summary>
        /// Полностью заменяет таблицу ldap_employees:
        /// TRUNCATE + BulkInsert.
        /// </summary>
        public async Task ReplaceAllLdapEmployees(List<LDAPEmployee> employees)
        {
            await _ctx.Database.ExecuteSqlRawAsync("TRUNCATE TABLE \"ldap_employees\" RESTART IDENTITY");
            var cfg = new BulkConfig { BatchSize = 1000, UseTempDB = true };
            await _ctx.BulkInsertAsync(employees, cfg);
        }

        /// <summary>
        /// Обновляет LoginAd в основной таблице Employees по совпадению email.
        /// </summary>
        public async Task SyncEmployeeLoginsByEmail()
        {
            var ldapList = await _ctx.LdapEmployees
                .Where(x => !string.IsNullOrEmpty(x.Email))
                .ToListAsync();

            var map = ldapList
                .GroupBy(e => e.Email!.ToLower())
                .ToDictionary(g => g.Key, g => g.First().Login);

            var employees = await _ctx.Employees
                .Where(e => !string.IsNullOrEmpty(e.Mail))
                .ToListAsync();

            foreach (var emp in employees)
            {
                var mail = emp.Mail!.ToLower();
                if (map.TryGetValue(mail, out var login))
                    emp.LoginAd = login;
            }

            await _ctx.SaveChangesAsync();
        }
    }
}



using System;
using System.Collections.Generic;
using System.DirectoryServices.Protocols;
using System.Linq;
using System.Net;
using DinDin.Models;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;

namespace DinDin.Services
{
    public class LdapEmployeeSyncService
    {
        private readonly ILogger<LdapEmployeeSyncService> _logger;
        private readonly string _server;
        private readonly int _port;
        private readonly string _bindDn;
        private readonly string _password;
        private readonly string _searchBase;
        private readonly string _filter;
        private readonly string[] _attributes;

        public LdapEmployeeSyncService(IConfiguration configuration, ILogger<LdapEmployeeSyncService> logger)
        {
            _logger = logger;
            _server = configuration["LDAPConfig:Server"] ?? throw new ArgumentNullException("Server");
            _port = int.TryParse(configuration["LDAPConfig:Port"], out var port) ? port : 389;
            _bindDn = configuration["LDAPConfig:BindDN"] ?? throw new ArgumentNullException("BindDN");
            _password = configuration["LDAPConfig:Password"] ?? throw new ArgumentNullException("Password");
            _searchBase = configuration["LDAPConfig:SearchBase"] ?? throw new ArgumentNullException("SearchBase");
            _filter = configuration["LDAPConfig:Filters:UserPerson:Code"] ?? "(objectClass=user)";
            _attributes = configuration.GetSection("LDAPConfig:Filters:UserPerson:Keys").Get<string[]>() ?? Array.Empty<string>();

            _logger.LogInformation("LDAP Config loaded: Server={Server}, Port={Port}, BindDN={BindDN}, SearchBase={SearchBase}",
                _server, _port, _bindDn, _searchBase);
        }

        public List<LDAPEmployee> GetLdapEmployees()
        {
            // Настройка подключения LDAP
            _logger.LogInformation("Connecting to LDAP server {Server}:{Port}...", _server, _port);
            var credentials = new NetworkCredential(_bindDn, _password);
            var identifier = new LdapDirectoryIdentifier(_server, _port);
            using var connection = new LdapConnection(identifier, credentials, AuthType.Negotiate);
            connection.SessionOptions.ProtocolVersion = 3;
            connection.Bind();
            _logger.LogInformation("LDAP bind successful");

            var employees = new List<LDAPEmployee>();
            const int pageSize = 1000;
            // Сегменты по первой букве/цифре sAMAccountName
            const string segments = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ";

            foreach (var segment in segments)
            {
                // Фильтр для сегмента: начинается с символа
                var segmentFilter = $"(&{_filter}(sAMAccountName={segment}*))";
                _logger.LogInformation("Segment filter: {Filter}", segmentFilter);

                byte[] cookie = Array.Empty<byte>();
                var pageControl = new PageResultRequestControl(pageSize);

                do
                {
                    var searchRequest = new SearchRequest(_searchBase, segmentFilter, SearchScope.Subtree, _attributes);
                    pageControl.Cookie = cookie;
                    searchRequest.Controls.Add(pageControl);

                    var response = (SearchResponse)connection.SendRequest(searchRequest);
                    // Обрабатываем записи
                    foreach (SearchResultEntry entry in response.Entries)
                    {
                        try
                        {
                            employees.Add(new LDAPEmployee(entry));
                        }
                        catch (Exception ex)
                        {
                            _logger.LogWarning(ex, "Failed to parse entry in segment {Segment}", segment);
                        }
                    }

                    // Обновляем cookie для следующей страницы
                    cookie = response.Controls
                        .OfType<PageResultResponseControl>()
                        .FirstOrDefault()?
                        .Cookie
                        ?? Array.Empty<byte>();

                    _logger.LogInformation("Segment {Segment}: loaded {Count} entries, total so far: {Total}",
                        segment, response.Entries.Count, employees.Count);

                } while (cookie.Length > 0);
            }

            _logger.LogInformation("🎯 Total records retrieved from LDAP: {Count}", employees.Count);
            return employees;
        }
    }
}

using System.DirectoryServices.Protocols;

namespace DinDin.Extensions;

public static class LdapExtensions
{
    public static string? GetString(this SearchResultEntry entry, string attributeName)
    {
        if (entry.Attributes.Contains(attributeName))
            return entry.Attributes[attributeName]?[0]?.ToString();

        return null;
    }

    public static string[]? GetStringArray(this SearchResultEntry entry, string attributeName)
    {
        if (entry.Attributes.Contains(attributeName))
        {
            return entry.Attributes[attributeName]?
                .Cast<object>()
                .Select(x => x.ToString())
                .Where(x => !string.IsNullOrWhiteSpace(x))
                .ToArray();
        }

        return null;
    }

    public static int GetInt(this SearchResultEntry entry, string attributeName)
    {
        var str = entry.GetString(attributeName);
        return int.TryParse(str, out int val) ? val : 0;
    }
}



{
  "ConnectionStrings": {
    "BpmCore": "Host=localhost;Port=5432;Username=postgres;Password=postgres;Database=bpmbase_local;SslMode=Disable;SearchPath=default"
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
      }
    }
  }

}

using Microsoft.EntityFrameworkCore;

namespace DinDin.Models;

public partial class BpmcoreContext : DbContext
{
    public BpmcoreContext()
    {
    }

    public BpmcoreContext(DbContextOptions<BpmcoreContext> options)
        : base(options)
    {
    }

    public virtual DbSet<Department> Departments { get; set; }

    public virtual DbSet<Employee> Employees { get; set; }
    
    public DbSet<LDAPEmployee> LdapEmployees { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        => optionsBuilder.UseNpgsql("Host=localhost;Port=5432;Database=bpmbase_local ;Username=postgres;Password=postgres")
                        .EnableSensitiveDataLogging();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.HasPostgresExtension("csd", "pgcrypto");

        modelBuilder.Entity<Department>(entity =>
        {
            entity.HasKey(e => e.Id).HasName("departments2_pkey");

            entity.ToTable("Departments", "public");

            entity.Property(e => e.Id).HasColumnName("id");
            entity.Property(e => e.Actual).HasColumnName("actual");
            entity.Property(e => e.ManagerId).HasColumnName("manager_id");
            entity.Property(e => e.Name).HasColumnName("name");
            entity.Property(e => e.ParentId).HasColumnName("parent_id");

            entity.HasOne(d => d.Parent).WithMany(p => p.InverseParent)
                .HasForeignKey(d => d.ParentId)
                .OnDelete(DeleteBehavior.SetNull)
                .HasConstraintName("departments2_parent_id_fkey");
        });

        modelBuilder.Entity<Employee>(entity =>
        {
            entity.HasKey(e => e.Id).HasName("employees_pkey");

            entity.ToTable("Employees", "public");

            entity.HasIndex(e => e.TabNumber, "employees_tab_number_key").IsUnique();

            entity.Property(e => e.Id)
                .UseIdentityColumn() // если ты хочешь оставить sequence
                .HasColumnName("id");

            entity.Property(e => e.DepId).HasColumnName("dep_id");
            entity.Property(e => e.DepName).HasColumnName("dep_name");
            entity.Property(e => e.Disabled).HasColumnName("disabled");
            entity.Property(e => e.IsFilial).HasColumnName("is_filial");
            entity.Property(e => e.IsManager).HasColumnName("is_manager");
            entity.Property(e => e.LocalPhone).HasColumnName("local_phone");
            entity.Property(e => e.Login).HasColumnName("login");
            entity.Property(e => e.Mail).HasColumnName("mail");
            entity.Property(e => e.ManagerTabNumber).HasColumnName("manager_tab_number");
            entity.Property(e => e.MobilePhone).HasColumnName("mobile_phone");
            entity.Property(e => e.Name).HasColumnName("name");
            entity.Property(e => e.Position).HasColumnName("position");
            entity.Property(e => e.StatusCode).HasColumnName("status_code");
            entity.Property(e => e.StatusDescription).HasColumnName("status_description");
            entity.Property(e => e.TabNumber).HasColumnName("tab_number");
            entity.Property(e => e.ParentDepId).HasColumnName("parent_dep_id");
            entity.Property(e => e.ParentDepName).HasColumnName("parent_dep_name");

            entity.HasOne(d => d.ManagerTabNumberNavigation).WithMany(p => p.InverseManagerTabNumberNavigation)
                .HasPrincipalKey(p => p.TabNumber)
                .HasForeignKey(d => d.ManagerTabNumber)
                .OnDelete(DeleteBehavior.SetNull)
                .HasConstraintName("employees_manager_tab_number_fkey");
        });
        
        modelBuilder.Entity<LDAPEmployee>().ToTable("ldap_employees");

        OnModelCreatingPartial(modelBuilder);
    }

    
    partial void OnModelCreatingPartial(ModelBuilder modelBuilder);
}


