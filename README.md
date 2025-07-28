using BpmBaseApi.Domain.Entities.AccessControl;
using BpmBaseApi.Domain.Entities.Common;
using BpmBaseApi.Domain.Entities.Employees;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Domain.Entities.Reference;

namespace BpmBaseApi.Persistence.Interfaces
{
    public interface IUnitOfWork
    {
        void Commit();
        // public ApplicationDbContext DbContext { get; }
        Task CommitAsync(CancellationToken cancellationToken);
        IJournaledGenericRepository<UserEntity> UserRepository { get; }
        IJournaledGenericRepository<RoleEntity> RoleRepository { get; }
        IJournaledGenericRepository<GroupEntity> GroupRepository { get; }
        IJournaledGenericRepository<UserGroupEntity> UserGroupRepository { get; }
        IJournaledGenericRepository<UserRoleEntity> UserRoleRepository { get; }
        IJournaledGenericRepository<GroupRoleEntity> GroupRoleRepository { get; }
        IJournaledGenericRepository<BlockEntity> BlockRepository { get; }
        IJournaledGenericRepository<ProcessDataEntity> ProcessDataRepository { get; }
        IJournaledGenericRepository<ProcessEntity> ProcessRepository { get; }
        IJournaledGenericRepository<ProcessTaskEntity> ProcessTaskRepository { get; }
        IJournaledGenericRepository<ProcessTaskHistoryEntity> ProcessTaskHistoryRepository { get; }
        //IJournaledGenericRepository<RequestSequenceEntity> RequestSequenceRepository { get; }
        IJournaledGenericRepository<RefProcessCategoryEntity> RefProcessCategoryRepository { get; }
        IJournaledGenericRepository<RefProcessEntity> RefProcessRepository { get; }
        IJournaledGenericRepository<RefInformationSystemEntity> RefInformationSystemRepository { get; }
        IJournaledGenericRepository<WorkSessionEntity> WorkSessionRepository { get; }
        IJournaledGenericRepository<WorkSessionLogEntity> WorkSessionLogRepository { get; }
        IJournaledGenericRepository<RequestNumberCounterEntity> RequestNumberCounterRepository { get; }
        IJournaledGenericRepository<MenuItemEntity> MenuItemRepository { get; }
        IJournaledGenericRepository<RoleMenuEntity> RoleMenuRepository { get; }
        IJournaledGenericRepository<RefBusinessObjectEntity> RefBusinessObjectRepository { get; }
        IJournaledGenericRepository<RefBusinessObjectAttributeEntity> RefBusinessObjectAttributeRepository { get; }
        IGenericRepositoryInt<EmployeeEntity> EmployeeIntRepository { get; }
        IGenericRepositoryString<DepartmentEntity> DepartmentIntRepository { get; }
        IJournaledGenericRepository<DelegationEntity> DelegationRepository { get; }
    }
}


using BpmBaseApi.Domain.Entities.AccessControl;
using BpmBaseApi.Domain.Entities.Common;
using BpmBaseApi.Domain.Entities.Employees;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Domain.Entities.Reference;
using BpmBaseApi.Persistence.Interfaces;
using Microsoft.Extensions.DependencyInjection;

namespace BpmBaseApi.Persistence
{
    public class UnitOfWork : IUnitOfWork
    {
        private readonly ApplicationDbContext applicationDbContext;
        private readonly IServiceProvider serviceProvider;

        public UnitOfWork(ApplicationDbContext applicationDbContext, IServiceProvider serviceProvider)
        {
            this.applicationDbContext = applicationDbContext ?? throw new ArgumentNullException(nameof(applicationDbContext));
            this.serviceProvider = serviceProvider ?? throw new ArgumentNullException(nameof(serviceProvider));
        }

        public void Commit()
        {
            applicationDbContext.SaveChanges();
        }

        public Task CommitAsync(CancellationToken cancellationToken)
        {
            return this.applicationDbContext.SaveChangesAsync(cancellationToken);
        }
        // public ApplicationDbContext DbContext => applicationDbContext;
        public IJournaledGenericRepository<UserEntity> UserRepository =>
            serviceProvider.GetRequiredService<IJournaledGenericRepository<UserEntity>>();

        public IJournaledGenericRepository<RoleEntity> RoleRepository =>
            serviceProvider.GetRequiredService<IJournaledGenericRepository<RoleEntity>>();

        public IJournaledGenericRepository<GroupEntity> GroupRepository =>
            serviceProvider.GetRequiredService<IJournaledGenericRepository<GroupEntity>>();

        public IJournaledGenericRepository<UserGroupEntity> UserGroupRepository =>
            serviceProvider.GetRequiredService<IJournaledGenericRepository<UserGroupEntity>>();

        public IJournaledGenericRepository<UserRoleEntity> UserRoleRepository =>
            serviceProvider.GetRequiredService<IJournaledGenericRepository<UserRoleEntity>>();

        public IJournaledGenericRepository<GroupRoleEntity> GroupRoleRepository =>
            serviceProvider.GetRequiredService<IJournaledGenericRepository<GroupRoleEntity>>();

        public IJournaledGenericRepository<BlockEntity> BlockRepository =>
            serviceProvider.GetRequiredService<IJournaledGenericRepository<BlockEntity>>();

        public IJournaledGenericRepository<ProcessEntity> ProcessRepository =>
            serviceProvider.GetRequiredService<IJournaledGenericRepository<ProcessEntity>>();

        public IJournaledGenericRepository<ProcessDataEntity> ProcessDataRepository =>
            serviceProvider.GetRequiredService<IJournaledGenericRepository<ProcessDataEntity>>();

        public IJournaledGenericRepository<ProcessTaskEntity> ProcessTaskRepository =>
            serviceProvider.GetRequiredService<IJournaledGenericRepository<ProcessTaskEntity>>();

        public IJournaledGenericRepository<ProcessTaskHistoryEntity> ProcessTaskHistoryRepository =>
            serviceProvider.GetRequiredService<IJournaledGenericRepository<ProcessTaskHistoryEntity>>();

        public IJournaledGenericRepository<RefProcessCategoryEntity> RefProcessCategoryRepository =>
            serviceProvider.GetRequiredService<IJournaledGenericRepository<RefProcessCategoryEntity>>();

        public IJournaledGenericRepository<RefProcessEntity> RefProcessRepository =>
            serviceProvider.GetRequiredService<IJournaledGenericRepository<RefProcessEntity>>();

        public IJournaledGenericRepository<RefInformationSystemEntity> RefInformationSystemRepository =>
           serviceProvider.GetRequiredService<IJournaledGenericRepository<RefInformationSystemEntity>>();

        public IJournaledGenericRepository<WorkSessionEntity> WorkSessionRepository =>
          serviceProvider.GetRequiredService<IJournaledGenericRepository<WorkSessionEntity>>();

        public IJournaledGenericRepository<WorkSessionLogEntity> WorkSessionLogRepository =>
          serviceProvider.GetRequiredService<IJournaledGenericRepository<WorkSessionLogEntity>>();

        public IJournaledGenericRepository<RequestNumberCounterEntity> RequestNumberCounterRepository =>
          serviceProvider.GetRequiredService<IJournaledGenericRepository<RequestNumberCounterEntity>>();

        public IJournaledGenericRepository<MenuItemEntity> MenuItemRepository =>
          serviceProvider.GetRequiredService<IJournaledGenericRepository<MenuItemEntity>>();

        public IJournaledGenericRepository<RoleMenuEntity> RoleMenuRepository =>
          serviceProvider.GetRequiredService<IJournaledGenericRepository<RoleMenuEntity>>();
        
        public IGenericRepositoryInt<EmployeeEntity> EmployeeIntRepository =>
            serviceProvider.GetRequiredService<IGenericRepositoryInt<EmployeeEntity>>();

        public IGenericRepositoryString<DepartmentEntity> DepartmentIntRepository =>
            serviceProvider.GetRequiredService<IGenericRepositoryString<DepartmentEntity>>();


        public IJournaledGenericRepository<RefBusinessObjectEntity> RefBusinessObjectRepository =>
            serviceProvider.GetRequiredService<IJournaledGenericRepository<RefBusinessObjectEntity>>();

        public IJournaledGenericRepository<RefBusinessObjectAttributeEntity> RefBusinessObjectAttributeRepository =>
            serviceProvider.GetRequiredService<IJournaledGenericRepository<RefBusinessObjectAttributeEntity>>();
        
        public IJournaledGenericRepository<DelegationEntity> DelegationRepository =>
            serviceProvider.GetRequiredService<IJournaledGenericRepository<DelegationEntity>>();
    }
}

