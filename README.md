using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using BpmBaseApi.Domain.Entities.Process;

namespace BpmBaseApi.Persistence.Interfaces
{
    /// <summary>
    /// Репозиторий для сущности делегаций, расширяющий общий CRUD из IJournaledGenericRepository
    /// </summary>
    public interface IDelegationRepository : IJournaledGenericRepository<DelegationEntity>
    {
        /// <summary>
        /// Возвращает все делегации, где заданный код — заместитель
        /// </summary>
        Task<List<DelegationEntity>> GetByDeputyAsync(string deputyCode, CancellationToken ct);
    }
}
