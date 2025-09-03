using AutoMapper;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Queries.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Application.QueryHandlers.Process
{
    public class GetProcessHistoryQueryHandler(
    IUnitOfWork unitOfWork,
    IMapper mapper,
    IProcessService processService
) : IRequestHandler<GetProcessHistoryQuery, BaseResponseDto<List<GetProcessHistoryResponse>>>
    {
        public async Task<BaseResponseDto<List<GetProcessHistoryResponse>>> Handle(GetProcessHistoryQuery query, CancellationToken cancellationToken)
        {
            var processTask = await unitOfWork.ProcessTaskRepository
                .GetByFilterAsync(cancellationToken, h => h.Id == query.TaskId);

            var history = await unitOfWork.ProcessTaskHistoryRepository
                .GetByFilterListAsync(cancellationToken, h => h.ProcessDataId == processTask.ProcessDataId);

            var historyItems = mapper.Map<List<GetProcessHistoryResponse>>(history.OrderBy(h => h.Timestamp));

            var tree = processService.BuildHistoryTree(historyItems);

            return new BaseResponseDto<List<GetProcessHistoryResponse>> { Data = tree };
        }
    }
}

using BpmBaseApi.Domain.Entities.AccessControl;
using BpmBaseApi.Domain.Entities.Common;
using BpmBaseApi.Domain.Entities.Employees;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Domain.Entities.Reference;
using static Dapper.SqlMapper;

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
        IJournaledGenericRepository<RefProcessStageEntity> RefProcessStageRepository { get; }
        IJournaledGenericRepository<RefDeliveryTypeEntity> RefDeliveryTypeRepository { get; }
        IJournaledGenericRepository<ProcessFileEntity> ProcessFileRepository { get; }
        IJournaledGenericRepository<RefIncomeSourceEntity> RefIncomeSourceRepository { get; }
        IJournaledGenericRepository<RefApplicantCategoryEntity> RefApplicantCategoryRepository { get; }
        IJournaledGenericRepository<RefCorrespondenceTypeEntity> RefCorrespondenceTypeRepository { get; }
        IJournaledGenericRepository<RefApprovalTypeEntity> RefApprovalTypeRepository { get; }
        IJournaledGenericRepository<RefSecurityClassificationEntity> RefSecurityClassificationRepository { get; }
        IJournaledGenericRepository<RefCatalogNumberEntity> RefCatalogNumberRepository { get; }
        IJournaledGenericRepository<RefCatalogFolderEntity> RefCatalogFolderRepository { get; }
        IJournaledGenericRepository<RefCorrespondenceNatureEntity> RefCorrespondenceNatureRepository { get; }
        IJournaledGenericRepository<RefRegisterDocumentEntity> RefRegisterDocumentRepository { get; }
        IJournaledGenericRepository<RefRegisterDocumentFolderEntity> RefRegisterDocumentFolderRepository { get; }
        IJournaledGenericRepository<SignFileEntity> SignFileRepository { get; }
    }
}
