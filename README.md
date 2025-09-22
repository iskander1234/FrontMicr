
BuildClassificationVariables(...)


camundaService.CamundaSubmitTask(new CamundaSubmitTaskRequest {
    TaskId = claimed.TaskId,
    Variables = stageVars   // здесь внутри уже есть classification = "Normal"
});


using System.Threading.Tasks;
using AutoMapper;
using BpmBaseApi.Domain.Models;
using BpmBaseApi.Services.Camunda.CamundaProvider.Models;
using BpmBaseApi.Services.Camunda.ServiceCamunda;
using BpmBaseApi.Services.Camunda.ServiceCamunda.Models;
using BpmBaseApi.Services.Interfaces;
using BpmBaseApi.Shared.Enum;
using BpmBaseApi.Shared.Models.Camunda;

namespace BpmBaseApi.Services.Camunda
{
    public class CamundaService
        (
        IMapper mapper,
        ICamundaProvider camundaProvider
        ) : ICamundaService
    {
        public async Task<CamundaClaimedTasksResponse> CamundaClaimedTasks(string processGuid, string stage, CancellationToken cancellationToken = default)
        {
            try
            {
                var result = await camundaProvider.ClaimedTasksAsync(processGuid, stage, cancellationToken);

                if (result.Count == 0)
                {
                    throw new HandlerException($"Не найдена задача в Camunda: {processGuid} {stage};");
                }

                var task = result.FirstOrDefault();
                
                return new CamundaClaimedTasksResponse()
                {
                    TaskId = task.id,
                    Name = task.name,
                };
            }
            catch (HandlerException)
            {
                throw;
            }
            catch (Exception ex)
            {
                throw new HandlerException($"Некорректный ответ от Camunda: {ex.Message} {ex.InnerException?.Message};");
            }
        }

        public async Task<string> CamundaStartProcess(CamundaStartProcessRequest request, CancellationToken cancellationToken)
        {
            var requestProvider = mapper.Map<StartProcessProviderRequest>(request);
            try
            {
                var result = await camundaProvider.StartProcessAsync(request.processCode, requestProvider, cancellationToken);
                return result;
            }
            catch (HandlerException)
            {
                throw;
            }
            catch (Exception ex)
            {
                throw new HandlerException($"Некорректный ответ от Camunda: {ex.Message} {ex.InnerException?.Message};");
            }
        }

        public async Task<CamundaSubmitTaskResponse> CamundaSubmitTask(CamundaSubmitTaskRequest request, CancellationToken cancellationToken)
        {
            try
            {
                var result = await camundaProvider.SubmitTaskAsync(request.TaskId, request.Variables, cancellationToken);

                return mapper.Map<CamundaSubmitTaskResponse>(result);
            }
            catch (HandlerException)
            {
                throw;
            }
            catch (Exception ex)
            {
                throw new HandlerException($"Некорректный ответ от Camunda: {ex.Message} {ex.InnerException?.Message};");
            }
        }
    }
}
