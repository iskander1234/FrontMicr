using System.Linq.Expressions;
using BpmBaseApi.Domain.SeedWork;

namespace BpmBaseApi.Persistence.Interfaces;

public interface IGenericRepositoryInt<TEntity>
{
    ValueTask<TEntity?> GetByFilterAsync(CancellationToken cancellationToken, Expression<Func<TEntity, bool>>? filter = null);

    ValueTask<List<TEntity>> GetByFilterListAsync(CancellationToken cancellationToken, Expression<Func<TEntity, bool>>? filter = null);

    Task<int> CountAsync(CancellationToken cancellationToken, Expression<Func<TEntity, bool>>? filter = null);

    ValueTask<TEntity?> GetByIdAsync(CancellationToken cancellationToken, int id);

    Task DeleteByIdAsync(int id, CancellationToken cancellationToken);
}


using System.Linq.Expressions;
using BpmBaseApi.Domain.SeedWork;
using BpmBaseApi.Persistence.Interfaces;
using Microsoft.EntityFrameworkCore;

namespace BpmBaseApi.Persistence.Repositories;

public class GenericRepositoryInt<TEntity> : IGenericRepositoryInt<TEntity>
    where TEntity : BaseIntEntity
{
    private readonly ApplicationDbContext _context;
    private readonly DbSet<TEntity> _dbSet;

    public GenericRepositoryInt(ApplicationDbContext context)
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

    public async ValueTask<TEntity?> GetByIdAsync(CancellationToken cancellationToken, int id)
    {
        return await _dbSet.FirstOrDefaultAsync(e => e.Id == id, cancellationToken);
    }

    public async Task DeleteByIdAsync(int id, CancellationToken cancellationToken)
    {
        var entity = await GetByIdAsync(cancellationToken, id);
        if (entity != null)
        {
            _dbSet.Remove(entity);
            await _context.SaveChangesAsync(cancellationToken);
        }
    }
}
