{
  "ConnectionStrings": {
    "BpmCore": "Host=localhost;Port=5432;Username=postgres;Password=postgres;Database=bpmbase7;SslMode=Disable;SearchPath=default"
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
      }
    }
  }
}
