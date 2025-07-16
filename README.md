"Ldap": {
  "Host": "ldap.enpf.kz",
  "Port": 389,
  "User": "CN=ldap_user,OU=Users,DC=enpf,DC=kz",
  "Password": "yourpassword",
  "SearchBase": "DC=enpf,DC=kz"
}

await SyncLdapAsync(serviceProvider);
