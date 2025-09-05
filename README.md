// Команда Save — по сути как Start, только без Camunda
using MediatR;
using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;

namespace BpmBaseApi.Shared.Commands.Process
{
    public record SaveProcessCommand(
        string ProcessCode,
        Dictionary<string, object> Payload
    ) : IRequest<BaseResponseDto<StartProcessResponse>>;
}
