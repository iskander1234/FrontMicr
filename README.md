[Table("ldap_employees")]

modelBuilder.Entity<LdapEmployee>().ToTable("ldap_employees");

public DbSet<LdapEmployee> LdapEmployees { get; set; }
