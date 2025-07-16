"LDAPConfig": {
  "Server": "172.31.0.252",
  "Port": "389",
  "BindDN": "CN=gnpfadm,CN=Users,DC=enpf,DC=kz",
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
