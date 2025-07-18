public abstract class BaseStringEntity
{
    public string Id { get; set; } = null!;
}


public class DepartmentEntity : BaseStringEntity

public interface IGenericRepositoryString<TEntity> : IGenericRepository<TEntity, string> where TEntity : BaseStringEntity
