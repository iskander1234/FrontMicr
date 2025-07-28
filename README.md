using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Persistence.Interfaces;

namespace BpmBaseApi.Persistence.Repositories;

public class DelegationRepository
    : JournaledGenericRepository<DelegationEntity>, IDelegationRepository
{
    public DelegationRepository(ApplicationDbContext ctx) : base(ctx) { }

    public Task<List<DelegationEntity>> GetByDeputyAsync(
        string deputyCode,
        CancellationToken ct) =>
        _dbSet
            .Where(d => d.DeputyUserCode.ToLower() == deputyCode.ToLower())
            .ToListAsync(ct);
}
