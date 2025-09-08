using System.Linq.Expressions;
using BpmBaseApi.Domain.SeedWork;

namespace BpmBaseApi.Persistence.Interfaces
{
    public interface IJournaledGenericRepository<TEntity>
    {
        ValueTask<TEntity?> GetByFilterAsync(CancellationToken cancellationToken, Expression<Func<TEntity, bool>>? filter = null);

        /// <summary>
        /// Getting entity by Id 
        /// </summary>
        /// <param name="id"></param>
        /// <param name="cancellationToken"></param>
        /// <returns></returns>
        ValueTask<List<TEntity>> GetByFilterListAsync(CancellationToken cancellationToken, Expression<Func<TEntity, bool>>? filter = null, Func<IQueryable<TEntity>, IQueryable<TEntity>>? include = null, Func<IQueryable<TEntity>, IOrderedQueryable<TEntity>>? orderBy = null);

        Task<int> CountAsync(CancellationToken cancellationToken, Expression<Func<TEntity, bool>>? filter = null);

        Task<Guid> RaiseEvent(BaseEntityEvent @event, CancellationToken cancellationToken, bool autoCommit = true);

        ValueTask<TEntity?> GetByIdAsync(CancellationToken cancellationToken, Guid id);

        Task<decimal> SumAsync(CancellationToken cancellationToken, Expression<Func<TEntity, decimal>> byParam, Expression<Func<TEntity, bool>>? filter = null);
        Task DeleteByIdAsync(Guid id, CancellationToken cancellationToken);
    }

}
Нет UpdateAsync
