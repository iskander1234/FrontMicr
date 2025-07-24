namespace DailyDataUpdateApp.Interfaces.Shared.Repositories
{
    internal interface IListRepository<T>
    {
        Task UpdateByKey(List<T> items);
    }
}



using Microsoft.EntityFrameworkCore.Metadata.Builders;
using System;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations.Schema;
using System.DirectoryServices.Protocols;
using System.Linq;
using System.Reflection;
using System.Text.Json.Serialization;

namespace DailyDataUpdateApp.Models
{
    [Serializable]
    public record LDAPEmployee : IEntityTypeConfiguration<LDAPEmployee>
    {
        [Column("id")]               
        public int    Id               { get; set; }

        [Column("user_status_code"), JsonPropertyName("UserAccountControl")]
        public int    UserStatusCode   { get; set; }

        [Column("is_disabled")]
        public bool   IsDisabled       { get; set; }

        [Column("given_name", TypeName="varchar"), JsonPropertyName("givenName")]
        public string?GivenName        { get; set; }

        [Column("name", TypeName="varchar"), JsonPropertyName("cn")]
        public string Name             { get; set; } = "";

        [Column("position_name", TypeName="varchar"), JsonPropertyName("description")]
        public string?PositionName    { get; set; }

        [Column("title", TypeName="varchar"), JsonPropertyName("title")]
        public string?Title           { get; set; }

        [Column("department", TypeName="varchar"), JsonPropertyName("department")]
        public string?Department      { get; set; }

        [Column("login", TypeName="varchar"), JsonPropertyName("sAMAccountName")]
        public string Login            { get; set; } = "";

        [Column("local_phone", TypeName="varchar"), JsonPropertyName("ipPhone")]
        public string?LocalPhone      { get; set; }

        [Column("mobile_number", TypeName="varchar"), JsonPropertyName("telephoneNumber")]
        public string?MobileNumber    { get; set; }

        [Column("city", TypeName="varchar"), JsonPropertyName("l")]
        public string?City            { get; set; }

        [Column("location_address", TypeName="varchar"), JsonPropertyName("streetAddress")]
        public string?LocationAddress { get; set; }

        [Column("postal_code", TypeName="varchar"), JsonPropertyName("postalCode")]
        public string?PostalCode      { get; set; }

        [Column("company", TypeName="varchar"), JsonPropertyName("company")]
        public string?Company         { get; set; }

        [Column("manager", TypeName="varchar"), JsonPropertyName("manager")]
        public string?Manager         { get; set; }

        [Column("email", TypeName="varchar"), JsonPropertyName("mail")]
        public string?Email           { get; set; }

        [Column("member_of"), JsonPropertyName("memberof")]
        public List<string>?MemberOf  { get; set; }

        [Column("member_of_string", TypeName="varchar")]
        public string?MemberOfString  { get; set; }

        [Column("is_manager")]
        public bool   IsManager       { get; set; }

        [Column("group_id")]
        public int?   GroupId         { get; set; }

        [Column("department_id")]
        public int?   DepartmentId    { get; set; }

        [Column("department_details", TypeName="jsonb")]
        public List<DepartmentDetail>? DepartmentDetails { get; set; }

        public LDAPEmployee() { }

        public LDAPEmployee(in SearchResultEntry entry)
        {
            foreach (var p in typeof(LDAPEmployee).GetProperties())
            {
                var json = p.GetCustomAttribute<JsonPropertyNameAttribute>()?.Name;
                if (json == null || !entry.Attributes.Contains(json)) continue;
                var raw = entry.Attributes[json][0]?.ToString();
                if (string.IsNullOrEmpty(raw)) continue;

                if (json == "memberof")
                {
                    MemberOfString = raw.Replace(",DC=enpf,DC=kz", "");
                    MemberOf = MemberOfString
                        .Replace(",OU=Группы","")
                        .Replace(",CN=Users","")
                        .Replace("OU=","")
                        .Replace("CN=","")
                        .Split(',')
                        .Distinct()
                        .ToList();
                }
                else if (p.PropertyType == typeof(int) && int.TryParse(raw, out var iv))
                {
                    if (json == "UserAccountControl")
                        IsDisabled = (iv & 0x0002) == 0x0002;
                    p.SetValue(this, iv);
                }
                else
                {
                    p.SetValue(this, raw);
                }
            }
        }

        public void Configure(EntityTypeBuilder<LDAPEmployee> builder)
        {
            builder.ToTable("ldap_employees","public", e => e.ExcludeFromMigrations());
            builder.HasKey(e => e.Id);
            builder.Property(e => e.Id).UseIdentityColumn();
            builder.HasIndex(e => e.Login).IsUnique();
        }
    }
}



using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using System;
using System.Collections.Generic;
using System.DirectoryServices.Protocols;
using System.Linq;
using System.Net;

namespace DailyDataUpdateApp.Services.Ldap
{
    internal abstract class LDAPService
    {
        private readonly LdapDirectoryIdentifier _id;
        private readonly NetworkCredential _cred;
        private readonly string[] _attrs;
        private readonly string _baseDn;
        protected abstract string Filter { get; }

        protected LDAPService(IConfiguration cfg, ILogger logger)
        {
            var srv = cfg["LDAPConfig:Server"]!;
            var port= int.TryParse(cfg["LDAPConfig:Port"], out var p)? p:389;
            _id    = new LdapDirectoryIdentifier(srv, port);
            _cred  = new NetworkCredential(cfg["LDAPConfig:BindDN"]!, cfg["LDAPConfig:Password"]!);
            _baseDn= cfg["LDAPConfig:SearchBase"]!;
            _attrs = cfg.GetSection("LDAPConfig:Filters:UserPerson:Keys").Get<string[]>()!;
            logger.LogInformation("LDAPService init {0}:{1}, Base={2}",srv,port,_baseDn);
        }

        protected IEnumerable<SearchResultEntry> LdapPagedSearch()
        {
            using var conn = new LdapConnection(_id, _cred, AuthType.Negotiate);
            conn.SessionOptions.ProtocolVersion = 3;
            conn.Bind();

            var all = new List<SearchResultEntry>();
            byte[] cookie;

            do
            {
                var req = new SearchRequest(_baseDn, Filter, SearchScope.Subtree, _attrs);
                req.Controls.Add(new PageResultRequestControl(1000) { Cookie = cookie });
                var resp = (SearchResponse)conn.SendRequest(req);
                all.AddRange(resp.Entries.Cast<SearchResultEntry>());
                cookie = resp.Controls
                              .OfType<PageResultResponseControl>()
                              .FirstOrDefault()?.Cookie
                         ?? Array.Empty<byte>();
            } while (cookie.Length > 0);

            return all;
        }
    }
}



using DailyDataUpdateApp.Interfaces.Complex.Services;
using DailyDataUpdateApp.Models;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using System.Collections.Generic;
using System.Linq;

namespace DailyDataUpdateApp.Services.Ldap
{
    internal class UserLDAPService : LDAPService, IUserLdapService
    {
        public UserLDAPService(IConfiguration cfg, ILogger<UserLDAPService> log)
            : base(cfg, log) { }

        protected override string Filter => "(&(objectClass=user)(objectCategory=PERSON))";

        public IEnumerable<LDAPEmployee> GetLDAPEmployees()
        {
            var entries = LdapPagedSearch();
            return entries.Select(e => new LDAPEmployee(e)).ToList();
        }

        public void PrintAttributes(in SearchResultEntry entry)
        {
            foreach (var name in entry.Attributes.AttributeNames.Cast<string>())
            {
                var val = entry.Attributes[name][0]?.ToString();
                Console.WriteLine($"{name} = {val}");
            }
        }
    }
}



using DailyDataUpdateApp.Interfaces.Shared.Repositories;
using DailyDataUpdateApp.Models;
using DailyDataUpdateApp.Shared.Contexes;
using EFCore.BulkExtensions;
using System.Collections.Generic;
using System.Threading.Tasks;

namespace DailyDataUpdateApp.Repositories
{
    internal class LDAPUsersRepository : IListRepository<LDAPEmployee>
    {
        private readonly BpmCoreUserContext _ctx;
        public LDAPUsersRepository(BpmCoreUserContext ctx) => _ctx = ctx;

        public async Task UpdateByKey(List<LDAPEmployee> list)
        {
            await _ctx.BulkInsertOrUpdateAsync(list, opts =>
            {
                opts.BatchSize          = 1000;
                opts.IncludeGraph       = false;
                opts.UpdateByProperties = new[] { "Login" };
                opts.SetOutputIdentity  = true;
                opts.UseTempDB          = false;
                opts.PropertiesToExclude= new[] { "Id","GroupId","DepartmentId","IsManager" };
            });
        }
    }
}
