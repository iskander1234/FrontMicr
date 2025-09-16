 /// <summary>
        /// Метод получения files из processData
        /// </summary>
        /// <param name="query">Запрос получения заявок по Id</param>
        /// <param name="cancellationToken">Маркер отмены, используемый для отмены HTTP-запроса.</param>
        /// <returns></returns>
        [HttpPost("getfilesbypd")]
        [SwaggerResponse(StatusCodes.Status200OK, "Данные успешно получены", typeof(BaseResponseDto<GetFilesByPdResponse>))]
        [SwaggerResponse(StatusCodes.Status404NotFound)]
        [SwaggerResponse(StatusCodes.Status400BadRequest)]
        [SwaggerResponse(StatusCodes.Status409Conflict)]
        [SwaggerResponse(StatusCodes.Status500InternalServerError)]
        [SwaggerResponse(StatusCodes.Status503ServiceUnavailable)]
        public async Task<IActionResult> GetFilesByPdAsync([FromBody] GetFilesByPdQuery query, CancellationToken cancellationToken)
        {
            var result = await mediator.Send(query, cancellationToken);

            return Ok(result);
        }

        using AutoMapper;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Models.Process;
using BpmBaseApi.Shared.Queries.Process;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Application.QueryHandlers.Process;


public class GetFilesByPdQueryHandler(
    IMapper mapper,
    IUnitOfWork unitOfWork
) : IRequestHandler<GetFilesByPdQuery, BaseResponseDto<GetFilesByPdResponse>>
{
    public async Task<BaseResponseDto<GetFilesByPdResponse>> Handle(GetFilesByPdQuery query, CancellationToken cancellationToken)
    {
        var process = await unitOfWork.ProcessDataRepository.GetByFilterAsync(cancellationToken, p => p.Id == query.ProcessId)
                     ?? throw new HandlerException("Запрос с таким идентификатором не найден", ErrorCodesEnum.Business);

        var processFiles = await unitOfWork
            .ProcessFileRepository
            .GetByFilterListAsync
            (
                cancellationToken,
                p => p.ProcessDataId == process.ProcessId 
                     && p.State == ProcessFileState.Active
            );

        var response = mapper.Map<GetFilesByPdResponse>(process);
       

        return new BaseResponseDto<GetFilesByPdResponse> { Data = response };
    }
}

using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Shared.Queries.Process;

public class GetFilesByPdQuery : IRequest<BaseResponseDto<GetFilesByPdResponse>> 
{
    /// <summary>
    /// Уникальный идентификатор задачи
    /// </summary>
    public Guid ProcessId { get; set; }
}

using BpmBaseApi.Shared.Models.Process;

namespace BpmBaseApi.Shared.Responses.Process;

public class GetFilesByPdResponse
{
    /// <summary>Идентификатор (UUID) файла в системе</summary>
    public string FileId { get; set; } = default!;

    /// <summary>Имя файла</summary>
    public string FileName { get; set; }  = default!;

    /// <summary>Тип или расширение файла</summary>
    public string FileType { get; set; }  = default!;
}


using AutoMapper;
using BpmBaseApi.Domain.Entities.AccessControl;
using BpmBaseApi.Domain.Entities.Employees;
using BpmBaseApi.Domain.Entities.Process;
using BpmBaseApi.Domain.Entities.Reference;
using BpmBaseApi.Shared.Models.Process;
using BpmBaseApi.Shared.Models.Reference;
using BpmBaseApi.Shared.Responses.AccessControl;
using BpmBaseApi.Shared.Responses.Employee;
using BpmBaseApi.Shared.Responses.Process;
using BpmBaseApi.Shared.Responses.Reference;

namespace BpmBaseApi.Application.Configurations
{
    public class MappingProfile : Profile
    {
        public MappingProfile()
        {
            CreateMap<UserRoleEntity, GetUserRolesResponse>();
            CreateMap<UserGroupEntity, GetUserGroupResponse>();
            CreateMap<MenuItemEntity, GetMenuItemResponse>()
    .ForMember(dest => dest.Children, opt => opt.Ignore());

            CreateMap<RefCatalogFolderEntity, CatalogFolder>();
            CreateMap<EmployeeEntity, GetEmployeeByTabNumberResponse>();

            CreateMap<RefProcessEntity, GetRefProcessResponse>();
            CreateMap<RefInformationSystemEntity, GetRefInformationSystemResponse>();
            CreateMap<RefBusinessObjectEntity, GetRefBusinessObjectResponse>();
            CreateMap<RefBusinessObjectAttributeEntity, BusinessObjectAttribute>();
            CreateMap<RefDeliveryTypeEntity, GetRefDeliveryTypeResponse>();
            CreateMap<RefIncomeSourceEntity, GetRefIncomeSourceResponse>();
            CreateMap<RefApplicantCategoryEntity, GetRefApplicantCategoryResponse>();
            CreateMap<RefCorrespondenceTypeEntity, GetRefCorrespondenceTypeResponse>();
            CreateMap<RefApprovalTypeEntity, GetRefApprovalTypeResponse>();
            CreateMap<RefSecurityClassificationEntity, GetRefSecurityClassificationResponse>();
            CreateMap<RefProcessStageEntity, GetRefProcessStageResponse>();
            CreateMap<RefCatalogNumberEntity, GetRefCatalogNumberResponse>();
            CreateMap<RefCorrespondenceNatureEntity, GetRefCorrespondenceNatureResponse>();

            CreateMap<ProcessTaskEntity, GetUserTasksResponse>();
           // CreateMap<ProcessTaskEntity, GetFilesByPdResponse>();
            CreateMap<ProcessTaskHistoryEntity, GetProcessHistoryResponse>();
            CreateMap<GetProcessHistoryResponse, GetApprovalProcessHistoryResponse>();
            CreateMap<GetProcessHistoryResponse, GetExecutionProcessHistoryResponse>();
            CreateMap<ProcessDataEntity, GetUserRelatedProcessesResponse>();

            CreateMap<ProcessFileEntity, FileDto>();
            CreateMap<ProcessFileEntity, GetFilesByPdResponse>();

            CreateMap<RefRegisterDocumentFolderEntity, GetRefRegisterDocumentFoldersResponse>();
            CreateMap<RefRegisterDocumentFolderEntity, RegisterDocumentFolder>();
            CreateMap<RefRegisterDocumentEntity, RegisterDocument>();

            CreateMap<RefGroupEntity, GetRefGroupResponse>();
            CreateMap<RefGroupUserEntity, GroupUser>();

            CreateMap<GetProcessHistoryResponse, GetProcessHistoryByPDResponse>();
        }
    }

}
