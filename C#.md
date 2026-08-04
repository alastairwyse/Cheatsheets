Covariance and Contra and C# page in cheatsheets
  Casting a generic type OR passing to a method


When to use a Shim class and example
  Unit testing something that doesn't provide an interface
  Unit testing extension methods
  (or should this belong in a patterns md)

C# object pool
  https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.objectpool.objectpool-1?view=netframework-4.8-pp
  https://learn.microsoft.com/en-us/aspnet/core/performance/objectpool?view=aspnetcore-10.0

C# lookup
  https://learn.microsoft.com/en-us/dotnet/api/system.linq.lookup-2?view=net-10.0


#### ExceptionDispatchInfo

It's not just used for passing exceptions between threads.  See below example from ApplicationAccess...

I add a public Exception property to the ApplicationAccess.Validation.ValidationResult which is used to convert exceptions thrown by a ConcurrentAccessManager instance into results of validation.   Then in the ApplicationAccess.Persistence.AccessManagerTemporalEventPersisterBufferBase class I adjust this method which rethrows a validation exception if it occurs...

```C#
/// <summary>
/// Re-throws the exception which caused validation failure, if the exception exists.
/// </summary>
/// <param name="validationResult">The validation result to check.</param>
protected void ThrowExceptionIfValidationFails(ValidationResult validationResult)
{
    /*
    if (validationResult.Successful == false && validationResult.ValidationExceptionDispatchInfo != null)
    {
        validationResult.ValidationExceptionDispatchInfo.Throw();
    }
    */
    if (validationResult.Successful == false && validationResult.ValidationException != null)
    {
        throw validationResult.ValidationException;
    }

}
```

Then if I run the solution and try to add the same user twice it results in this exception stack...

```
fail: Microsoft.AspNetCore.Diagnostics.ExceptionHandlerMiddleware[1]
      An unhandled exception has occurred while executing the request.
      System.ArgumentException: User 'TestUser' already exists. (Parameter 'user')
       ---> ApplicationAccess.LeafVertexAlreadyExistsException`1[System.String]: Vertex 'TestUser' already exists in the graph.
         at ApplicationAccess.DirectedGraphBase`2.AddLeafVertex(TLeaf leafVertex) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess\DirectedGraphBase.cs:line 131
         at ApplicationAccess.ConcurrentDirectedGraph`2.<>n__1(TLeaf leafVertex)
         at ApplicationAccess.ConcurrentDirectedGraph`2.<>c__DisplayClass20_0.<AddLeafVertex>b__1() in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess\ConcurrentDirectedGraph.cs:line 218
         at ApplicationAccess.Metrics.MetricLoggingConcurrentDirectedGraph`2.<AddLeafVertex>b__13_0(TLeaf actionLeaf, Action baseAction) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Metrics\MetricLoggingConcurrentDirectedGraph.cs:line 125
         at ApplicationAccess.ConcurrentDirectedGraph`2.<>c__DisplayClass20_0.<AddLeafVertex>b__0() in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess\ConcurrentDirectedGraph.cs:line 218
         at ApplicationAccess.ConcurrentDirectedGraph`2.AcquireLocksAndInvokeAction(Object lockObject, LockObjectDependencyPattern lockObjectDependencyPattern, Action action) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess\ConcurrentDirectedGraph.cs:line 335
         at ApplicationAccess.ConcurrentDirectedGraph`2.AddLeafVertex(TLeaf leafVertex, Action`2 wrappingAction) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess\ConcurrentDirectedGraph.cs:line 220
         at ApplicationAccess.Metrics.MetricLoggingConcurrentDirectedGraph`2.AddLeafVertex(TLeaf leafVertex) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Metrics\MetricLoggingConcurrentDirectedGraph.cs:line 128
         at ApplicationAccess.AccessManagerBase`4.AddUser(TUser user) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess\AccessManagerBase.cs:line 126
         --- End of inner exception stack trace ---
         at ApplicationAccess.Persistence.AccessManagerTemporalEventPersisterBufferBase`4.ThrowExceptionIfValidationFails(ValidationResult validationResult) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Persistence\AccessManagerTemporalEventPersisterBufferBase.cs:line 672
         at ApplicationAccess.Persistence.AccessManagerTemporalEventPersisterBufferBase`4.<>c__DisplayClass36_0.<AddUser>b__0() in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Persistence\AccessManagerTemporalEventPersisterBufferBase.cs:line 210
         at ApplicationAccess.Utilities.LockManager.AcquireLocksAndInvokeAction(List`1 lockObjects, Int32 nextObjectIndex, Action action) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Utilities\LockManager.cs:line 254         at ApplicationAccess.Utilities.LockManager.AcquireLocksAndInvokeAction(List`1 lockObjects, Int32 nextObjectIndex, Action action) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Utilities\LockManager.cs:line 260         at ApplicationAccess.Utilities.LockManager.AcquireLocksAndInvokeAction(Object lockObject, LockObjectDependencyPattern lockObjectDependencyPattern, Action action) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Utilities\LockManager.cs:line 181
         at ApplicationAccess.Persistence.AccessManagerTemporalEventPersisterBufferBase`4.AddUser(TUser user) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Persistence\AccessManagerTemporalEventPersisterBufferBase.cs:line 208
         at ApplicationAccess.Hosting.ReaderWriterNodeBase`5.AddUser(TUser user) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Hosting\ReaderWriterNodeBase.cs:line 324
         at ApplicationAccess.Hosting.Rest.Controllers.AddPrimaryUserEventProcessorControllerBase.AddUser(String user) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Hosting.Rest\Controllers\AddPrimaryUserEventProcessorControllerBase.cs:line 55
         at lambda_method3(Closure, Object, Object[])
         at Microsoft.AspNetCore.Mvc.Infrastructure.ActionMethodExecutor.SyncActionResultExecutor.Execute(ActionContext actionContext, IActionResultTypeMapper mapper, ObjectMethodExecutor executor, Object controller, Object[] arguments)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.InvokeActionMethodAsync()
         at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.Next(State& next, Scope& scope, Object& state, Boolean& isCompleted)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.InvokeNextActionFilterAsync()
      --- End of stack trace from previous location ---
         at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.Rethrow(ActionExecutedContextSealed context)         at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.Next(State& next, Scope& scope, Object& state, Boolean& isCompleted)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.InvokeInnerFilterAsync()
      --- End of stack trace from previous location ---
         at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeFilterPipelineAsync>g__Awaited|20_0(ResourceInvoker invoker, Task lastTask, State next, Scope scope, Object state, Boolean isCompleted)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeAsync>g__Awaited|17_0(ResourceInvoker invoker, Task task, IDisposable scope)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeAsync>g__Awaited|17_0(ResourceInvoker invoker, Task task, IDisposable scope)
         at ApplicationAccess.Hosting.Rest.TripSwitchMiddleware.<InitializeSwitchNotActuatedAction>b__9_0(HttpContext context) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Hosting.Rest\TripSwitchMiddleware.cs:line 159
         at ApplicationAccess.Hosting.Rest.TripSwitchMiddleware.InvokeAsync(HttpContext context) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Hosting.Rest\TripSwitchMiddleware.cs:line 146
         at Microsoft.AspNetCore.Diagnostics.ExceptionHandlerMiddlewareImpl.<Invoke>g__Awaited|10_0(ExceptionHandlerMiddlewareImpl middleware, HttpContext context, Task task)
```

However, if I revert the code to use the ExceptionDispatchInfo class to hold the exception, and then use that classes' Throw() method to rethrown...

```C#
/// <summary>
/// Re-throws the exception which caused validation failure, if the exception exists.
/// </summary>
/// <param name="validationResult">The validation result to check.</param>
protected void ThrowExceptionIfValidationFails(ValidationResult validationResult)
{
    if (validationResult.Successful == false && validationResult.ValidationExceptionDispatchInfo != null)
    {
        validationResult.ValidationExceptionDispatchInfo.Throw();
    }
}
```

... and then run the same test adding an exising user, I get this exception stack...

```
fail: Microsoft.AspNetCore.Diagnostics.ExceptionHandlerMiddleware[1]
      An unhandled exception has occurred while executing the request.
      System.ArgumentException: User 'TestUser' already exists. (Parameter 'user')
       ---> ApplicationAccess.LeafVertexAlreadyExistsException`1[System.String]: Vertex 'TestUser' already exists in the graph.
         at ApplicationAccess.DirectedGraphBase`2.AddLeafVertex(TLeaf leafVertex) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess\DirectedGraphBase.cs:line 131
         at ApplicationAccess.ConcurrentDirectedGraph`2.<>n__1(TLeaf leafVertex)
         at ApplicationAccess.ConcurrentDirectedGraph`2.<>c__DisplayClass20_0.<AddLeafVertex>b__1() in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess\ConcurrentDirectedGraph.cs:line 218
         at ApplicationAccess.Metrics.MetricLoggingConcurrentDirectedGraph`2.<AddLeafVertex>b__13_0(TLeaf actionLeaf, Action baseAction) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Metrics\MetricLoggingConcurrentDirectedGraph.cs:line 125
         at ApplicationAccess.ConcurrentDirectedGraph`2.<>c__DisplayClass20_0.<AddLeafVertex>b__0() in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess\ConcurrentDirectedGraph.cs:line 218
         at ApplicationAccess.ConcurrentDirectedGraph`2.AcquireLocksAndInvokeAction(Object lockObject, LockObjectDependencyPattern lockObjectDependencyPattern, Action action) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess\ConcurrentDirectedGraph.cs:line 335
         at ApplicationAccess.ConcurrentDirectedGraph`2.AddLeafVertex(TLeaf leafVertex, Action`2 wrappingAction) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess\ConcurrentDirectedGraph.cs:line 220
         at ApplicationAccess.Metrics.MetricLoggingConcurrentDirectedGraph`2.AddLeafVertex(TLeaf leafVertex) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Metrics\MetricLoggingConcurrentDirectedGraph.cs:line 128
         at ApplicationAccess.AccessManagerBase`4.AddUser(TUser user) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess\AccessManagerBase.cs:line 126
         --- End of inner exception stack trace ---
         at ApplicationAccess.AccessManagerBase`4.AddUser(TUser user) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess\AccessManagerBase.cs:line 130
         at ApplicationAccess.ConcurrentAccessManager`4.<>n__1(TUser user)
         at ApplicationAccess.ConcurrentAccessManager`4.<>c__DisplayClass51_0.<AddUser>b__1() in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess\ConcurrentAccessManager.cs:line 664
         at ApplicationAccess.Metrics.ConcurrentAccessManagerMetricLogger`4.<>c__DisplayClass14_1.<GenerateAddUserMetricLoggingWrappingAction>b__2() in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Metrics\ConcurrentAccessManagerMetricLogger.cs:line 122
         at ApplicationAccess.ConcurrentAccessManager`4.<>c__DisplayClass10_0.<AddUser>b__0(TUser actionUser, Action baseAction) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess\ConcurrentAccessManager.cs:line 104
         at ApplicationAccess.Metrics.ConcurrentAccessManagerMetricLogger`4.<>c__DisplayClass14_1.<GenerateAddUserMetricLoggingWrappingAction>b__1() in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Metrics\ConcurrentAccessManagerMetricLogger.cs:line 120
         at ApplicationAccess.Metrics.ConcurrentAccessManagerMetricLogger`4.CallAccessManagerEventProcessingMethodWithMetricLogging[TIntervalMetric,TCountMetric](Action eventAction) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Metrics\ConcurrentAccessManagerMetricLogger.cs:line 1737
         at ApplicationAccess.Metrics.ConcurrentAccessManagerMetricLogger`4.<>c__DisplayClass14_0.<GenerateAddUserMetricLoggingWrappingAction>b__0(TUser metricLoggingActionUser, Action baseAction) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Metrics\ConcurrentAccessManagerMetricLogger.cs:line 116
         at ApplicationAccess.ConcurrentAccessManager`4.<>c__DisplayClass51_0.<AddUser>b__0() in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess\ConcurrentAccessManager.cs:line 664
         at ApplicationAccess.Utilities.LockManager.AcquireLocksAndInvokeAction(List`1 lockObjects, Int32 nextObjectIndex, Action action) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Utilities\LockManager.cs:line 254
         at ApplicationAccess.Utilities.LockManager.AcquireLocksAndInvokeAction(List`1 lockObjects, Int32 nextObjectIndex, Action action) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Utilities\LockManager.cs:line 260
         at ApplicationAccess.Utilities.LockManager.AcquireLocksAndInvokeAction(Object lockObject, LockObjectDependencyPattern lockObjectDependencyPattern, Action action) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Utilities\LockManager.cs:line 181
         at ApplicationAccess.ConcurrentAccessManager`4.AddUser(TUser user, Action`2 wrappingAction) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess\ConcurrentAccessManager.cs:line 662
         at ApplicationAccess.Metrics.MetricLoggingConcurrentAccessManager`4.AddUser(TUser user, Action`2 wrappingAction) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Metrics\MetricLoggingConcurrentAccessManager.cs:line 262
         at ApplicationAccess.ConcurrentAccessManager`4.AddUser(TUser user, Action`1 postProcessingAction) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess\ConcurrentAccessManager.cs:line 107
         at ApplicationAccess.Validation.ConcurrentAccessManagerEventValidator`4.<>c__DisplayClass4_0.<ValidateAddUser>b__0() in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Validation\ConcurrentAccessManagerEventValidator.cs:line 53
         at ApplicationAccess.Validation.ConcurrentAccessManagerEventValidator`4.InvokeActionAndWrapResponse(Action concurrentAccessManagerAction) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Validation\ConcurrentAccessManagerEventValidator.cs:line 181
      --- End of stack trace from previous location ---
         at ApplicationAccess.Persistence.AccessManagerTemporalEventPersisterBufferBase`4.ThrowExceptionIfValidationFails(ValidationResult validationResult) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Persistence\AccessManagerTemporalEventPersisterBufferBase.cs:line 666
         at ApplicationAccess.Persistence.AccessManagerTemporalEventPersisterBufferBase`4.<>c__DisplayClass36_0.<AddUser>b__0() in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Persistence\AccessManagerTemporalEventPersisterBufferBase.cs:line 210
         at ApplicationAccess.Utilities.LockManager.AcquireLocksAndInvokeAction(List`1 lockObjects, Int32 nextObjectIndex, Action action) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Utilities\LockManager.cs:line 254
         at ApplicationAccess.Utilities.LockManager.AcquireLocksAndInvokeAction(List`1 lockObjects, Int32 nextObjectIndex, Action action) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Utilities\LockManager.cs:line 260
         at ApplicationAccess.Utilities.LockManager.AcquireLocksAndInvokeAction(Object lockObject, LockObjectDependencyPattern lockObjectDependencyPattern, Action action) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Utilities\LockManager.cs:line 181
         at ApplicationAccess.Persistence.AccessManagerTemporalEventPersisterBufferBase`4.AddUser(TUser user) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Persistence\AccessManagerTemporalEventPersisterBufferBase.cs:line 208
         at ApplicationAccess.Hosting.ReaderWriterNodeBase`5.AddUser(TUser user) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Hosting\ReaderWriterNodeBase.cs:line 324
         at ApplicationAccess.Hosting.Rest.Controllers.AddPrimaryUserEventProcessorControllerBase.AddUser(String user) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Hosting.Rest\Controllers\AddPrimaryUserEventProcessorControllerBase.cs:line 55
         at lambda_method3(Closure, Object, Object[])
         at Microsoft.AspNetCore.Mvc.Infrastructure.ActionMethodExecutor.SyncActionResultExecutor.Execute(ActionContext actionContext, IActionResultTypeMapper mapper, ObjectMethodExecutor executor, Object controller, Object[] arguments)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.InvokeActionMethodAsync()
         at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.Next(State& next, Scope& scope, Object& state, Boolean& isCompleted)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.InvokeNextActionFilterAsync()
      --- End of stack trace from previous location ---
         at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.Rethrow(ActionExecutedContextSealed context)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.Next(State& next, Scope& scope, Object& state, Boolean& isCompleted)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ControllerActionInvoker.InvokeInnerFilterAsync()
      --- End of stack trace from previous location ---
         at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeFilterPipelineAsync>g__Awaited|20_0(ResourceInvoker invoker, Task lastTask, State next, Scope scope, Object state, Boolean isCompleted)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeAsync>g__Awaited|17_0(ResourceInvoker invoker, Task task, IDisposable scope)
         at Microsoft.AspNetCore.Mvc.Infrastructure.ResourceInvoker.<InvokeAsync>g__Awaited|17_0(ResourceInvoker invoker, Task task, IDisposable scope)
         at ApplicationAccess.Hosting.Rest.TripSwitchMiddleware.<InitializeSwitchNotActuatedAction>b__9_0(HttpContext context) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Hosting.Rest\TripSwitchMiddleware.cs:line 159
         at ApplicationAccess.Hosting.Rest.TripSwitchMiddleware.InvokeAsync(HttpContext context) in C:\Development\Multi-Language\ApplicationAccess\ApplicationAccess.Hosting.Rest\TripSwitchMiddleware.cs:line 146
         at Microsoft.AspNetCore.Diagnostics.ExceptionHandlerMiddlewareImpl.<Invoke>g__Awaited|10_0(ExceptionHandlerMiddlewareImpl middleware, HttpContext context, Task task)
```

Notice that the second exception stack is longer and includes references to class ConcurrentAccessManagerEventValidator which aren't included in the first.