PS C:\BPM\bpm\bpmbaseapi> dotnet ef migrations add AddDelegationsTable --project BpmBaseApi.Persistence --startup-project BpmBaseApi
Build started...
Build succeeded.
Unable to create a 'DbContext' of type 'ApplicationDbContext'. The exception 'No suitable constructor was found for entit
y type 'DelegationEntity'. The following constructors had parameters that could not be bound to properties of the entity type:
    Cannot bind 'principalCode', 'principalName', 'deputyCode', 'deputyName' in 'DelegationEntity(string principalCode, string principalName, string deputyCode, string deputyName)'
Note that only mapped properties can be bound to constructor parameters. Navigations to related entities, including refer
ences to owned types, cannot be bound.' was thrown while attempting to create an instance. For the different patterns supported at design time, see https://go.microsoft.com/fwlink/?linkid=851728
PS C:\BPM\bpm\bpmbaseapi> 
