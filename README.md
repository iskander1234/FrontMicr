using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Persistence;
using BpmBaseApi.Persistence.Interfaces;
using Microsoft.EntityFrameworkCore;

namespace BpmBaseApi.Persistence.Implementations
{
    public class DelegationRepository
        : JournaledGenericRepository<DelegationEntity>, IDelegationRepository
    {
        public DelegationRepository(BpmBaseApiDbContext ctx) : base(ctx) { }

        public Task<List<DelegationEntity>> GetByDeputyAsync(
            string deputyCode,
            CancellationToken ct) =>
            _dbSet
                .Where(d => d.DeputyUserCode.ToLower() == deputyCode.ToLower())
                .ToListAsync(ct);
    }
}
