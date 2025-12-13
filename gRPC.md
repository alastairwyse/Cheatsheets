### Lessons Learned from gRPC Implemenetation in [ApplicationAccess](https://github.com/alastairwyse/ApplicationAccess)

#### Proto Files

Writing proto files is fairly straightforward.  The [Microsoft](https://learn.microsoft.com/en-us/aspnet/core/grpc/basics?view=aspnetcore-8.0#proto-file) and [Google](https://grpc.io/docs/what-is-grpc/introduction/) intro documentation give examples which are easy to follow.  You can follow good separation of concerns practices by keeping the proto model definitions separate from the RPC definitions (in different files), and then using imports e.g...

```proto
import "Models/Protos/temporal_event_buffer_items.proto";
```

#### Imports, Building, and Project Structure

At a high level, follow the [Codegen Integration Into .NET Build](https://chromium.googlesource.com/external/github.com/grpc/grpc/+/HEAD/src/csharp/BUILD-INTEGRATION.md#getting-started) documentation to enable building the gRPC generated code.  Basically you need to...

1. Import the ['Grpc.Tools' NuGet package](https://www.nuget.org/packages/grpc.tools/), and include something similar to the below in the .csproj file 'PackageReference' definition...

```XML
<PackageReference Include="Grpc.Tools" Version="2.72.0">
  <PrivateAssets>all</PrivateAssets>
  <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
</PackageReference>
```

2. Add references to the .proto files that you want to generate code for.  These proto files can be linked-to in a different project, i.e. using 'ProtoRoot' and 'Link' directives as per below, which again allows good separation of concerns practices...

```XML
<ItemGroup>
  <Protobuf Include="..\ApplicationAccess.Hosting.Grpc\Protos\event_cache_v1.proto" ProtoRoot="..\ApplicationAccess.Hosting.Grpc" Link="Protos\event_cache_v1.proto" GrpcServices="Client" />
</ItemGroup>
```

A few gotchas and hints to make things work smoothly...

* Make sure that each proto file is only referenced by a single client project (i.e. using the 'Protobuf Include' syntax described above) once for each of 'GrpcServices' = 'Client' or 'Server'.  I.e. in the above case of 'event_cache_v1.proto', only one of each of the below declarations should occur within all the .csproj files of the solution...

```XML
<Protobuf Include="..\ApplicationAccess.Hosting.Grpc\Protos\event_cache_v1.proto" ProtoRoot="..\ApplicationAccess.Hosting.Grpc" Link="Protos\event_cache_v1.proto" GrpcServices="Client" />
<Protobuf Include="..\ApplicationAccess.Hosting.Grpc\Protos\event_cache_v1.proto" ProtoRoot="..\ApplicationAccess.Hosting.Grpc" Link="Protos\event_cache_v1.proto" GrpcServices="Server" />
```

Reason is that duplicate declarations will cause gRPC to generate two instances of the same code, which although they are from the same source, .NET recognizes and different, but conflicting complied code.  This may not trip you up so much in the runtime client and server code (as they'd typically exist in separate namespaces and/or projects), but in things like integration tests where you may well import both it can become a problem.  The typical symptom is an error message like below when trying to declare a variable of conflicting type...

```
CS0436: The type 'X' in '{namespace Y}' conflicts with the imported type 'Y' in '{namespace Z}'.
```

* Don't put the gRPC generated code into the same namespace as the classes which are referencing/consuming it.  This can often lead to the same CS0436 conflict errors described above, and can also cause naming conflicts between the gRPC generated classes and client classes/namespaces.  E.g. (in the case of the ApplicationAccess gRPC EventCache) the proto file would contain a definition for a service called 'EventCache'...

```proto
service EventCache {
	rpc GetAllEventsSince (etc...)
```

...if you build/generate the C# code from this proto file into the ApplicationAccess.Hosting.Grpc.EventCache namespace/project which contains the server-side implementation of the EventCache, the generated service class name is the same as the namespace... which leads to compiler ambiguities and failures as the project gets referenced and expanded.

A better approach is to have all the generated code created in it's own dedicated namespace.  This can be achieved by setting a 'package' declaration in the proto file like below...

```proto
package ApplicationAccess.Hosting.Grpc.GeneratedCode.EventCache.V1;
```

...then all the generated code is created once in its own dedicated namespace, and can be selectively imported into client code just as any non-generated class library could.

For the EventCache, I ended up with the following 4 projects/namespaces, which followed clean separation of concerns and avoided the issues discussed above...

| Namespace / Project | Contents | Notes |
| ------------------- | -------- | ----- |
| [ApplicationAccess.Hosting.Grpc](https://github.com/alastairwyse/ApplicationAccess/tree/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Grpc) | All proto files and common gRPC-related utility classes | Contains a separate folder (i.e. and namespace) for 'Models', so that gRPC services/RPCs and models can be defined separately. |
| ApplicationAccess.Hosting.Grpc.GeneratedCode.EventCache.V1 | Generated gRPC classes | Doesn't actually exist in the solution (no source code), but is generated at build-time by the Grpc.Tools package |
| [ApplicationAccess.Hosting.Grpc.Client](https://github.com/alastairwyse/ApplicationAccess/tree/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Grpc.Client) | The EventCache gRPC client class (and soon other gRPC client classes) | Contains [this .proto file reference](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Grpc.Client/ApplicationAccess.Hosting.Grpc.Client.csproj#L29) to generate the client-side gRPC interfaces...<br /> ``` <Protobuf Include="..\ApplicationAccess.Hosting.Grpc\Protos\event_cache_v1.proto" ProtoRoot="..\ApplicationAccess.Hosting.Grpc" Link="Protos\event_cache_v1.proto" GrpcServices="Client" /> ``` <br /> Consuming code [imports the generated gRPC code](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Grpc.Client/EventCacheClient.cs#L25) from namespace ApplicationAccess.Hosting.Grpc.GeneratedCode.EventCache.V1 |
| [ApplicationAccess.Hosting.Grpc.EventCache](https://github.com/alastairwyse/ApplicationAccess/tree/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Grpc.EventCache) | The EventCache gRPC service (hosted in ASP.NET core) | Contains [this .proto file reference](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Grpc.EventCache/ApplicationAccess.Hosting.Grpc.EventCache.csproj#L34) to generate the client-side gRPC interfaces...<br /> ``` <Protobuf Include="..\ApplicationAccess.Hosting.Grpc\Protos\event_cache_v1.proto" ProtoRoot="..\ApplicationAccess.Hosting.Grpc" Link="Protos\event_cache_v1.proto" GrpcServices="Server" /> ``` <br /> Consuming code [imports the generated gRPC code](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Grpc.EventCache/EventCacheService.cs#L23) from namespace ApplicationAccess.Hosting.Grpc.GeneratedCode.EventCache.V1 |

#### API Versioning

AFAIK gRPC doesn't offer a native/internal API versioning mechanism like you have with Swagger and REST.  But, you can define different versions of services/RPCs by defining separate .proto files for each, and having them build into separate packages/namespaces (where the version is included in the namespace).  I did this using the 'package' declaration in the .proto file...

```proto
package ApplicationAccess.Hosting.Grpc.GeneratedCode.EventCache.V1;
```

At the end of the day this approach isn't massively dissimilar to Swagger... in Swagger/REST you'd differentiate your versions using the request URL, query parameter, or HTTP header, but each version of an endpoint would still need to route to distinct .NET code via a Controller method... and you could argue that the code underlying these controller methods should be differentiated using namespaces which include the version... not unlike the 'package' definition above.

#### Error Handling

gRPC error handling is quite well [documented for .NET](https://learn.microsoft.com/en-us/aspnet/core/grpc/error-handling?view=aspnetcore-8.0).  By default, exceptions on the server side are caught, and serialized and passed to the client as [RpcException](https://grpc.github.io/grpc/csharp/api/Grpc.Core.RpcException.html) objects.  In addition to a message (i.e. [the one included in standard Exceptions](https://learn.microsoft.com/en-us/dotnet/api/system.exception.message?view=net-8.0)), RpcException includes a [status code](https://grpc.github.io/grpc/core/md_doc_statuscodes.html) property.  This includes statuses similar to the commmon ones used in REST (INVALID_ARGUMENT, NOT_FOUND, PERMISSION_DENIED, UNAVAILABLE), plus others.

However in ApplicationAccess, I wanted the ability to send additional detail of exceptions and include the inner exception hierarchy from server to client, to match the equivalent REST-based functionality implemented in [ExceptionToHttpErrorResponseConverter](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Rest.Utilities/ExceptionToHttpErrorResponseConverter.cs) and [MiddlewareUtilities.SetupExceptionHandler](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Rest/MiddlewareUtilities.cs#L59).  gRPC offers [rich error handling](https://learn.microsoft.com/en-us/aspnet/core/grpc/error-handling?view=aspnetcore-8.0#rich-error-handling) for this purpose, which lets you include a custom protobuf object in the exception data passed from server to client.

I created protobuf object [GrpcError](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Grpc/Models/Protos/grpc_error.proto) as a gRPC equivalent to the [HttpErrorResponse](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Models/HttpErrorResponse.cs) used in the REST case...

```proto
message GrpcError {
	string code = 1;
	string message = 2;
	string target = 3;
	map<string, string> attributes = 4;
	GrpcError inner_error = 5;
}
```

The custom GrpcError needs to be wrapped in a Google.Rpc.Status object (which includes a status code and message similar to the RpcException object used in standard error handling), and then this converted to an RpcException to pass the rich exception from the server-side.  This is [clearly covered in the documentation](https://learn.microsoft.com/en-us/aspnet/core/grpc/error-handling?view=aspnetcore-8.0#creating-rich-errors-on-the-server), and implemented in ApplicationAccess in the [ExceptionToGrpcStatusConverter](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Grpc/ExceptionToGrpcStatusConverter.cs) and [ExceptionHandlingInterceptor](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Grpc/ExceptionHandlingInterceptor.cs) classes...

```C#
// From ExceptionHandlingInterceptor...

public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>
(
    TRequest request,
    ServerCallContext context,
    UnaryServerMethod<TRequest, TResponse> continuation
)
{
    try
    {
        return await continuation(request, context);
    }
    catch (Exception e)
    {
        logger.Log(LogLevel.Warning, e.Message, e);
        Google.Rpc.Status grpcStatus = exceptionToGrpcStatusConverter.Convert(e);
        if (grpcStatus.Code == (Int32)Code.Internal && errorHandlingOptions.OverrideInternalServerErrors.Value == true)
        {
            var overrideGrpcError = new GrpcError
            {
                Code = e.GetType().Name,
                Message = errorHandlingOptions.InternalServerErrorMessageOverride
            };
            var overrideStatus = new Google.Rpc.Status
            {
                Code = (Int32)Code.Internal,
                Message = errorHandlingOptions.InternalServerErrorMessageOverride,
                Details = { Google.Protobuf.WellKnownTypes.Any.Pack(overrideGrpcError) }
            };

            throw overrideStatus.ToRpcException();
        }
        else
        {
            throw grpcStatus.ToRpcException();
        }
    }
}
```

#### Interceptors

[Interceptors](https://learn.microsoft.com/en-us/aspnet/core/grpc/interceptors?view=aspnetcore-8.0) are the gRPC equivalent of custom ASP.NET middleware.  I use these in 2 places in ApplicationAccess, to create gRPC equivalents of REST functionality that was implemented using [custom middleware](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/write?view=aspnetcore-8.0)...

| Interceptor Class | Middleware Equivalent | Function |
| ----------------- | --------------------- | -------- |
| [ExceptionHandlingInterceptor](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Grpc/ExceptionHandlingInterceptor.cs) | [MiddlewareUtilities.SetupExceptionHandler()](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Rest/MiddlewareUtilities.cs#L59) | Intercepts any thrown .NET Exception, and converts it to an [RpcException](https://grpc.github.io/grpc/csharp/api/Grpc.Core.RpcException.html) which can be transported via gRPC.  Client classes then rehydrate and rethrow the original .NET Exception. |
| [TripSwitchInterceptor](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Grpc/TripSwitchInterceptor.cs) | [TripSwitchMiddleware](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Rest/TripSwitchMiddleware.cs) | Intercepts any incoming requests and either throws a defined exception or shuts down the service/application, if the tripswitch has been tripped/actuated |

Interceptors are setup during ASP.NET initialization when [calling the AddGrpc() method](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Grpc/ApplicationInitializer.cs#L133)...

```C#
builder.Services.AddGrpc(options =>
{
    options.Interceptors.Add<ExceptionHandlingInterceptor>();
    if (parameters.TripSwitchTrippedException != null)
    {
        options.Interceptors.Add<TripSwitchInterceptor>();
    }
});
```

#### Interceptors as Singletons

By default Interceptor classes have a per-request scope/lifetime... i.e. by default an ExceptionHandlingInterceptor class (added above) would be created for every new request.  But for the ExceptionHandlingInterceptor this doesn't work as is, since it needs needs to hold configuration... i.e. mappings from .NET Exceptions to RpcExceptions.  These are set via constructor parameters to the ExceptionHandlingInterceptor class, so you could register the parameters as services and have DI automatically pass them to the constructor, BUT there's no value re-instantiating a ExceptionHandlingInterceptor with every request when it could just be created once.  So if you [register a singleton instance of an Interceptor class](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Grpc/ApplicationInitializer.cs#L118) (before Add()ing it), gRPC will use that instance, rather than recreating.  I.e...

```
ExceptionHandlingInterceptor exceptionHandlingInterceptor = new(errorHandlingOptions, exceptionToGrpcStatusConverter);
builder.Services.AddSingleton<ExceptionHandlingInterceptor>(exceptionHandlingInterceptor);
```

#### Integration Tests

There is quite detailed [Microsoft documentation](https://learn.microsoft.com/en-us/aspnet/core/grpc/test-services?view=aspnetcore-8.0) and a [sample project](https://github.com/dotnet/AspNetCore.Docs/tree/main/aspnetcore/grpc/test-services/sample/Tests/Server/IntegrationTests) for this, although I found it a bit difficult to follow (quite a hierarchy of 'fixture' and 'helper' classes), and ultimately wanted to try and keep my integration test code inline with the REST-based integration tests as much as possible.

I found that you can utilize the [WebApplicationFactory](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests?view=aspnetcore-8.0&pivots=nunit#basic-tests-with-the-default-webapplicationfactory) class to create and run an in-memory ASP.NET web application, and then call the CreateClient() method on the resulting [TestServer](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.testhost.testserver?view=aspnetcore-8.0) object to get a HttpClient which will connect to that web application (this is what I do for the ApplicationAccess REST integration tests).  This is implemented in the [IntegrationTestsBase](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Grpc.EventCache.IntegrationTests/IntegrationTestsBase.cs) class.

The tricky hurdle to overcome is that obtaining the HttpClient via the CreateClient() alone will give you a HttpClient, but that client will fail with gRPC requests.  To overcome, you have to set the [HttpMessageHandler](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.httpmessagehandler?view=net-8.0) via the [GrpcChannelOptions](https://grpc.github.io/grpc/csharp-dotnet/api/Grpc.Net.Client.GrpcChannelOptions.html) when creating the gRPC client.  The HttpMessageHandler is obtained from the TestServer in the [IntegrationTestsBase.OneTimeSetUp()](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Grpc.EventCache.IntegrationTests/IntegrationTestsBase.cs#L52) method...

```C#
httpClient = testEventCache.CreateClient();
httpHandler = testEventCache.Server.CreateHandler();
GrpcChannelOptions channelOptions = new GrpcChannelOptions()
{
    HttpHandler = httpHandler
};
grpcClient = new EventCacheClient<String, String, String, String>
(
    new Uri(httpClient.BaseAddress.ToString()),
    channelOptions,
    userStringifier,
    groupStringifier,
    applicationComponentStringifier,
    accessLevelStringifier
);
grpcChannel = GrpcChannel.ForAddress(new Uri(httpClient.BaseAddress.ToString()), channelOptions);
```

Then the channel needs to be passed to the constructor of the gRPC client class that's generated from the RPC/service proto file.  Example below from the [EventCacheClient.GetAllEventsSince()](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Grpc.Client/EventCacheClient.cs#L142) method...

```C#
var client = new EventCacheRpc.EventCacheRpcClient(channel);
```

Sample integration test methods [here](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Grpc.EventCache.IntegrationTests/RpcTests.cs).

#### Kubernetes Health Checks

gRPC has its own defined [health checking protocol](https://github.com/grpc/grpc/blob/master/doc/health-checking.md), and implementing this in C# is again [well documented](https://learn.microsoft.com/en-us/aspnet/core/grpc/health-checks?view=aspnetcore-8.0).  To implement application-specific checks you need to provide an implementation of [IHealthCheck](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.diagnostics.healthchecks.ihealthcheck).  In the case of ApplicationAccess, the application health can be simply determined by checking whether the tripswitch has tripped/actuated or not... the [TripSwitchHealthCheck](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Grpc/TripSwitchHealthCheck.cs) class performs this.

To then register TripSwitchHealthCheck within ASP.NET I did the following (from the [ApplicationInitializer](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Grpc/ApplicationInitializer.cs) class)...

1. Instantiate a TripSwitchHealthCheck (need to pass the actuator to the constructor) and register it in DI as a singleton (for the same reason as described for [Interceptors as Singletons](#interceptors-as-singletons) as above... if you want the IHealthCheck implementation to have longer than per-request scope/lifetime, you need to register an instance of the class as a singleton)...

```C#
// Add gRPC health checks (using the TripSwitch to report the health)
TripSwitchHealthCheck tripSwitchHealthCheck = new(tripSwitchActuator);
builder.Services.AddSingleton<TripSwitchHealthCheck>(tripSwitchHealthCheck);
```

2. Call AddGrpcHealthChecks()...

```C#
builder.Services.AddGrpcHealthChecks().AddCheck<TripSwitchHealthCheck>("TripSwitch actuated health check");
```

3. Call MapGrpcHealthChecksService()...

```C#
app.MapGrpcHealthChecksService();
```

There was one issue with this in the ApplicationAccess context... if I remember correctly, the health check seemed to be registered in the request pipeline downstream of my custom Interceptors, including the [TripSwitchInterceptor](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Grpc/TripSwitchInterceptor.cs).  This meant that once the tripswitch was tripped/actuated, the TripSwitchInterceptor would intercept the health check request and throw its configured 'when tripped' exception, rather than reponding negatively to the health check request inline with the gRPC protocol.  To overcome this, I modified TripSwitchInterceptor to attempt to identify whether the incoming request was a health check request, and if so bypass the trip swicth functionality, passing to the next handler in the request pipeline.  Identifying the request as a health check request is done via the prefix of the RPC method name, which is subject to external change, but this works for the time being.  Extracts from the [TripSwitchInterceptor](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Hosting.Grpc/TripSwitchInterceptor.cs) class below...

```C#
/// <summary>The gRPC health checking protocol service definition package name prefix.</summary>
protected const String grpcHealthPackagePrefix = "grpc.health";

{...}

/// <summary>
/// Checks whether the specified name of the RPC method being called is a gRPC health check method. 
/// </summary>
/// <param name="rpcMethodName">The name of the RPC method being called.</param>
/// <returns>True if the method is a gRPC health check method.  False otherwise.</returns>
protected Boolean MethodIsHealthCheckMethod(String rpcMethodName)
{
    if (rpcMethodName.Length < grpcHealthPackagePrefix.Length + 1)
    {
        return false;
    }
    if (rpcMethodName.Substring(1, grpcHealthPackagePrefix.Length) == grpcHealthPackagePrefix)
    {
        return true;
    }
    else
    {
        return false;
    }
}

{...}

public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>
(
    TRequest request,
    ServerCallContext context,
    UnaryServerMethod<TRequest, TResponse> continuation
)
{
    if (MethodIsHealthCheckMethod(context.Method) == false)
    {
        // Tripswitch implementation removed
    }
    else
    {
        return await continuation(request, context);
    }
}

```

This health check responds to [gRPC probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-grpc-liveness-probe) in Kubernetes.

#### Running without HTTPS / TLS

In ApplicationAccess, all endpoints are exposed via plain HTTP, not HTTPS.  This is a deliberate design decision, as transport encryption is considered a concern of hosting infrastructure rather than the application (and many good infrastructure-based solutions are available, e.g. HTTPS/TLS termination via a Kubernetes Ingress).

gRPC services in ASP.NET Core can be run without HTTPS (e.g. by providing a plain HTTP URL to the 'urls' parameter on startup), but may result in the following exception when the gRPC service is called...

```
Error starting gRPC call. HttpRequestException: The HTTP/2 server closed the connection. HTTP/2 error code 'HTTP_1_1_REQUIRED' (0xd).
```

Assuming Kestral is used for hosting the service, this can be resolved by adding the below configuration to appsettings.json...

```XML
"Kestrel": {
  "EndpointDefaults": {
    "Protocols": "Http2"
  }
}
```

#### Transient Error Handling

TODO.  See https://learn.microsoft.com/en-us/aspnet/core/grpc/retries?view=aspnetcore-8.0
