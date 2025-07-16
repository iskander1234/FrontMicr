"LDAPConfig": {
    "Server": "",
    "Port": "389",
    "BindDN": "",
    "Password": "",
    "SearchBase": "",
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
