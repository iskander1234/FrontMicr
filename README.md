
        // 👇 Зарегистрировать ваш LDAP-сервис и интерфейс
        .AddSingleton<DinDin.Interface.IUserLdapService, DinDin.Services.UserLDAPService>()
        // Репозиторий
