using System.Linq.Expressions;

namespace BpmBaseApi.Persistence.Interfaces;

public interface IGenericRepositoryString<TEntity>
{
    ValueTask<TEntity?> GetByFilterAsync(CancellationToken cancellationToken, Expression<Func<TEntity, bool>>? filter = null);

    ValueTask<List<TEntity>> GetByFilterListAsync(CancellationToken cancellationToken, Expression<Func<TEntity, bool>>? filter = null);

    Task<int> CountAsync(CancellationToken cancellationToken, Expression<Func<TEntity, bool>>? filter = null);

    ValueTask<TEntity?> GetByIdAsync(CancellationToken cancellationToken, string id);

    Task DeleteByIdAsync(string id, CancellationToken cancellationToken);
}


using System.Linq.Expressions;
using BpmBaseApi.Persistence.Interfaces;
using Microsoft.EntityFrameworkCore;

namespace BpmBaseApi.Persistence.Repositories;

public class GenericRepositoryString<TEntity> : IGenericRepositoryString<TEntity>
    where TEntity : class
{
    private readonly ApplicationDbContext _context;
    private readonly DbSet<TEntity> _dbSet;

    public GenericRepositoryString(ApplicationDbContext context)
    {
        _context = context;
        _dbSet = _context.Set<TEntity>();
    }

    public async ValueTask<TEntity?> GetByFilterAsync(CancellationToken cancellationToken, Expression<Func<TEntity, bool>>? filter = null)
    {
        return await _dbSet.FirstOrDefaultAsync(filter, cancellationToken);
    }

    public async ValueTask<List<TEntity>> GetByFilterListAsync(CancellationToken cancellationToken, Expression<Func<TEntity, bool>>? filter = null)
    {
        var query = _dbSet.AsQueryable();
        if (filter != null) query = query.Where(filter);
        return await query.ToListAsync(cancellationToken);
    }

    public async Task<int> CountAsync(CancellationToken cancellationToken, Expression<Func<TEntity, bool>>? filter = null)
    {
        return await (filter != null ? _dbSet.CountAsync(filter, cancellationToken) : _dbSet.CountAsync(cancellationToken));
    }

    public async ValueTask<TEntity?> GetByIdAsync(CancellationToken cancellationToken, string id)
    {
        return await _dbSet.FindAsync(new object?[] { id }, cancellationToken: cancellationToken);
    }

    public async Task DeleteByIdAsync(string id, CancellationToken cancellationToken)
    {
        var entity = await GetByIdAsync(cancellationToken, id);
        if (entity != null)
        {
            _dbSet.Remove(entity);
            await _context.SaveChangesAsync(cancellationToken);
        }
    }
}




services.AddScoped(typeof(IGenericRepositoryString<>), typeof(GenericRepositoryString<>));





private readonly IGenericRepositoryString<DepartmentEntity> _departmentRepository;

public MyService(IGenericRepositoryString<DepartmentEntity> departmentRepository)
{
    _departmentRepository = departmentRepository;
}

