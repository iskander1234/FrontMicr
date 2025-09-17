using BpmBaseApi.Shared.Dtos;
using BpmBaseApi.Shared.Responses.Process;
using MediatR;

namespace BpmBaseApi.Shared.Commands.Process
{
    public class StartProcessCommand : IRequest<BaseResponseDto<StartProcessResponse>>
    {
        /// <summary>
        /// Код процесса
        /// </summary>
        public string ProcessCode { get; set; }

        /// <summary>
        /// Код инициатора
        /// </summary>
        public string InitiatorCode { get; set; }

        /// <summary>
        /// ФИО инициатора
        /// </summary>
        public string? InitiatorName { get; set; }
        
        /// <summary>
        /// Данные формы/входные параметры процесса
        /// </summary>
        public Dictionary<string, object>? Payload { get; set; }
    }
}
