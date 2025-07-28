fail: Microsoft.AspNetCore.Diagnostics.DeveloperExceptionPageMiddleware[1]
      An unhandled exception has occurred while executing the request.
      System.ArgumentException: GenericArguments[0], 'BpmBaseApi.Domain.Entities.Process.DelegationEntity', on 'BpmBaseApi.Persistence.Repositories.JournaledGenericRepository`1[TEntity]' violates the constraint of type 'TEntity'.
       ---> System.TypeLoadException: GenericArguments[0], 'BpmBaseApi.Domain.Entities.Process.DelegationEntity', on 'BpmBaseApi.Persistence.Repositories.JournaledGenericRepository`1[TEntity]' violates the constraint of type parameter 'TEntity'.
         at System.RuntimeTypeHandle.Instantiate(QCallTypeHandle handle, IntPtr* pInst, Int32 numGenericArgs, ObjectHandleOnStack type)
         at System.RuntimeType.MakeGenericType(Type[] instantiation)
         --- End of inner exception stack trace ---
         at System.RuntimeType.ValidateGenericArguments(MemberInfo definition, RuntimeType[] genericArguments, Exception e)
         at System.RuntimeType.MakeGenericType(Type[] instantiation)
         at Microsoft.Extensions.DependencyInjection.ServiceLookup.CallSiteFactory.TryCreateOpenGeneric(ServiceDescriptor descriptor, ServiceIdentifier serviceIdentifier, CallSiteChain callSiteChain, Int32 slot, Boolean throwOnConstraintViolation)
         at Microsoft.Extensions.DependencyInjection.ServiceLookup.CallSiteFactory.TryCreateOpenGeneric(ServiceIdentifier serviceIdentifier, CallSiteChain callSiteChain)
         at Microsoft.Extensions.DependencyInjection.ServiceLookup.CallSiteFactory.CreateCallSite(ServiceIdentifier serviceIdentifier, CallSiteChain callSiteChain)
         at Microsoft.Extensions.DependencyInjection.ServiceProvider.CreateServiceAccessor(ServiceIdentifier serviceIdentifier)
         at Microsoft.Extensions.DependencyInjection.ServiceProviderServiceExtensions.GetRequiredService(IServiceProvider provider, Type serviceType)
         at Microsoft.Extensions.DependencyInjection.ServiceProviderServiceExtensions.GetRequiredService[T](IServiceProvider provider)
         at BpmBaseApi.Persistence.UnitOfWork.get_DelegationRepository() in C:\BPM\bpm\bpmbaseapi\BpmBaseApi.Persistence\UnitOfWork.cs:line 103
         at BpmBaseApi.Application.CommandHandlers.Process.CreateDelegationCommandHandler.Handle(CreateDelegationCommand command, CancellationToken cancellationToken) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi.Application\CommandHandlers\Process\CreateDelegationCommandHandler.cs:line 47
         at BpmBaseApi.Controllers.V1.ProcessController.CreateAsync(CreateDelegationCommand command, CancellationToken cancellationToken) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi\Controllers\V1\ProcessController.cs:line 252
         at Microsoft.AspNetCore.Mvc.Infrastructure.ActionMethodExecutor.TaskOfIActionResultExecutor.Execute(ActionContext actionContext, IActionResultTypeMapper mapper, ObjectMethodExecutor executor, Object controller, Object[] arguments)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.<InvokeActionMethodAsync>g__Logged|12_1(ControllerActionInvoker invoker)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.<InvokeNextActionFilterAsync>g__Awaited|10_0(ControllerActionInvoker invoker, Task lastTask, State next, Scope scope, Object state, Boolean isCompleted)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.Rethrow(ActionExecutedContextSealed context)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.Next(State& next, Scope& scope, Object& state, Boolean& isCompleted)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.<InvokeInnerFilterAsync>g__Awaited|13_0(ControllerActionInvoker invoker, Task lastTask, State next, Scope scope, Object state, Boolean isCompleted)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeFilterPipelineAsync>g__Awaited|20_0(ResourceInvoker invoker, Task lastTask, State next, Scope scope, Object state, Boolean isCompleted)    
         at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeAsync>g__Logged|17_1(ResourceInvoker invoker)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeAsync>g__Logged|17_1(ResourceInvoker invoker)
         at BpmBaseApi.Middlewares.ErrorHandlerMiddleware.Invoke(HttpContext context) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi\Middlewares\ErrorHandlerMiddleware.cs:line 24
         at BpmBaseApi.Middlewares.DefineUserHandler.Invoke(HttpContext context) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi\Middlewares\DefineUserHandler.cs:line 30
         at BpmBaseApi.Middlewares.CorrelationMiddleware.Invoke(HttpContext context) in C:\BPM\bpm\bpmbaseapi\BpmBaseApi\Middlewares\CorrelationMiddleware.cs:line 16
         at Microsoft.AspNetCore.Authorization.AuthorizationMiddleware.Invoke(HttpContext context)
         at Microsoft.AspNetCore.Authentication.AuthenticationMiddleware.Invoke(HttpContext context)
         at Swashbuckle.AspNetCore.ReDoc.ReDocMiddleware.Invoke(HttpContext httpContext)
         at Swashbuckle.AspNetCore.SwaggerUI.SwaggerUIMiddleware.Invoke(HttpContext httpContext)
         at Swashbuckle.AspNetCore.Swagger.SwaggerMiddleware.Invoke(HttpContext httpContext, ISwaggerProvider swaggerProvider)
         at Microsoft.AspNetCore.Diagnostics.DeveloperExceptionPageMiddlewareImpl.Invoke(HttpContext context)

