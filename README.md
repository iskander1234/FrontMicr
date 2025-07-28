dotnet ef migrations add AddDelegationsTable --project BpmBaseApi.Persistence --startup-project BpmBaseApi.WebAPI
dotnet ef database update --project BpmBaseApi.Persistence --startup-project BpmBaseApi.WebAPI
