// Shared/Responses/Process/GetDeputiesResponse.cs
namespace BpmBaseApi.Shared.Responses.Process
{
    /// <summary>
    /// Ответ: данные одного заместителя
    /// </summary>
    public class GetDeputiesResponse
    {
        /// <summary>Код заместителя</summary>
        public string DeputyUserCode { get; set; }

        /// <summary>Имя заместителя</summary>
        public string DeputyUserName { get; set; }
    }
}


// Shared/Queries/Process/GetDeputiesQuery.cs
using MediatR;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;

namespace BpmBaseApi.Shared.Queries.Process
{
    /// <summary>
    /// Запрос: получить список заместителей для указанного пользователя (principalCode)
    /// </summary>
    public class GetDeputiesQuery : IRequest<BaseResponseDto<List<GetDeputiesResponse>>>
    {
        /// <summary>Код пользователя, для которого ищем заместителей</summary>
        public string PrincipalCode { get; set; }
    }
}


// Application/QueryHandlers/Process/GetDeputiesQueryHandler.cs
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using MediatR;
using BpmBaseApi.Persistence.Interfaces;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Queries.Process;
using BpmBaseApi.Shared.Responses.Process;

namespace BpmBaseApi.Application.QueryHandlers.Process
{
    public class GetDeputiesQueryHandler
        : IRequestHandler<GetDeputiesQuery, BaseResponseDto<List<GetDeputiesResponse>>>
    {
        private readonly IUnitOfWork _uow;

        public GetDeputiesQueryHandler(IUnitOfWork uow)
        {
            _uow = uow;
        }

        public async Task<BaseResponseDto<List<GetDeputiesResponse>>> Handle(
            GetDeputiesQuery query,
            CancellationToken cancellationToken)
        {
            // 1) фильтруем все записи делегаций, где я — PRINCIPAL
            var delegations = await _uow.DelegationRepository
                .GetByFilterListAsync(
                    cancellationToken,
                    d => d.PrincipalUserCode.ToLower() == query.PrincipalCode.ToLower()
                );

            // 2) проецируем в DTO
            var response = delegations
                .Select(d => new GetDeputiesResponse
                {
                    DeputyUserCode = d.DeputyUserCode,
                    DeputyUserName = d.DeputyUserName
                })
                .ToList();

            // 3) возвращаем через общий BaseResponseDto
            return new BaseResponseDto<List<GetDeputiesResponse>>
            {
                Data = response
            };
        }
    }
}



/// <summary>
        /// Получить список заместителей для указанного пользователя (principalCode).
        /// </summary>
        /// <param name="principalCode">Код пользователя, для которого ищем заместителей.</param>
        [HttpGet("deputies")]
        [ProducesResponseType(typeof(BaseResponseDto<List<GetDeputiesResponse>>), StatusCodes.Status200OK)]
        [ProducesResponseType(StatusCodes.Status400BadRequest)]
        public async Task<IActionResult> GetDeputies(
            [FromQuery] string principalCode,
            CancellationToken cancellationToken)
        {
            if (string.IsNullOrWhiteSpace(principalCode))
                return BadRequest(new BaseResponseDto<List<GetDeputiesResponse>>
                {
                    Message = "PrincipalCode не должен быть пустым",
                    ErrorCode = StatusCodes.Status400BadRequest
                });

            var query = new GetDeputiesQuery { PrincipalCode = principalCode };
            var result = await _mediator.Send(query, cancellationToken);
            return Ok(result);
        }
    }
