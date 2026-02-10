## REST API Design

This is more of a best practice guide rather than a cheatsheet. It describes the design practices and features which should be considered/included when designing a consistent, robust, and reliable REST API interface.

### Intro
According to the [original Roy Fielding paper introducing REST](https://roy.gbiv.com/pubs/dissertation/rest_arch_style.htm), REST is described as an 'architectural style for distributed hypermedia', and that paradigm can be adapted very nicely to provide interfaces for data in applications (i.e. a [resource model](https://www.thoughtworks.com/insights/blog/rest-api-design-resource-modeling))... i.e. where the URL provides a path to / definition of / hierarchy of the data, and the HTTP methods map to [CRUD operations](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete)...

| CRUD Operation | HTTP Method |
| -------------- | --------- |
| Create | PUT/POST |
| Read | GET |
| Update | PUT/PATCH |
| Delete | DELETE |

Where REST doesn’t adapt so nicely is with application functions which _don’t_ map to available HTTP methods... if you’re trying to map functions like 'convert', 'process', 'validate', etc... to REST, it’s somewhat like pushing the proverbial square peg into a round hole (although perhaps a square hole with rounded corners is a better analogy... some parts fit, but some don’t).. and this leads to one of the key principles when designing a REST interface... unless you’re only mapping CRUD operations on data, you’ll have to deviate from the REST principles and/or find creative ways to make your use case fit with REST (and particularly with HTTP methods)... and in these cases, **consistency is key**... i.e. decide on a way to implement a particular feature which steps outside REST principles, and then make sure similar features are implemented in the same way.


### URL Design

#### CRUD Operations
As mentioned above, CRUD operations are easy to implement.  [ApplicationAccess](https://github.com/alastairwyse/ApplicationAccess) exposes endpoints for CRUD operations on its primary data elements as follows...

| URL Path | Method | Description |
| /users/{user} | POST | Adds a user. |
| /users/{user} | DELETE | Removes a user. |
| /users/{user} | GET | Returns the specified user if it exists. |
| /users | GET | Returns all users. |
| /groups/{group} | POST | Adds a group. |
| /groups/{group} | DELETE | Removes a group. |
| /groups/{group} | GET | Returns the specified group if it exists. |
| /groups | GET | Returns all groups. |
| /entityTypes/{entityType} | POST | Adds an entity type. |
| /entityTypes/{entityType} | DELETE | Removes an entity type. |
| /entityTypes/{entityType} | GET | Returns the specified entity type if it exists. |
| /entityTypes | GET | Returns all entity types. |
| /entityTypes/{entityType}/entities/{entity} | POST | Adds an entity. |
| /entityTypes/{entityType}/entities/{entity} | DELETE | Removes an entity. |
| /entityTypes/{entityType}/entities/{entity} | GET | Returns the specified entity if it exists. |
| /entityTypes/{entityType}/entities | GET | Returns all entities of the specified type. |

#### Data Identifying and Filtering

The distinction between these two endpoints...

| URL Path | Method | Description |
| -------- | ------ | ----------- |
| /users/{user} | GET | Returns the specified user if it exists. |
| /users | GET | Returns all users. |

...raises an interesting question over how to filter data when querying.  In the simple case above where I just want to apply a single (and primary key) filter criteria (i.e. just specify the user to return), it made sense to use a URL which matched the equivalent POST and DELETE endpoints.  To filter by multiple criteria, I lean towards either of the below methods...

**Name/Value Pairs in the URL**

ApplicationAccess uses this approach for secondary data elements, where the name of the resource / data element appears first in the URL, followed by name value pairs which filter the data elements returned...

| URL Path | Method | Description |
| -------- | ------ | ----------- |
| /groupToEntityMappings/entityType/{entityType}/entity/{entity} | GET | Gets the groups that are mapped to the specified entity. |

This works well for equality filters for single values, but if other comparison types (&gt;, &lt;, etc...) or multiple values need to be specified you can consider...

**Query String**

Using a query string to filter data gives greater flexibility.  If you want to define multiple or varying filters, you could use a standard URL query string similar to...

```
/groupToEntityMappings?entityType=Clients&entity=CompanyA
```

...or...

```
/groupToEntityMappings?group=SalesStaff
```

You can also devise your own filtering/naming scheme, but instead it's often better to use an open standard, like [OData](https://www.odata.org/).  An OData filter on the 'groupToEntityMappings' resource described above could look like this...

```
/groupToEntityMappings?filter=(entityType eq 'Clients' and (entity eq 'CompanyA' or entity eq 'CompanyB'))
```

The plus of open standards like this is that you've got a greater chance or your clients being able to consume your API more easily... i.e. if their client software already supports OData.  The downside with OData is that is can be complex to parse and process the filter strings on the API server side.

It may make sense to use a hybrid of methods.  In ApplicationAccess I use a combination of filtering in the URL and querystring for certain endpoints... for example...

```
/userToGroupMappings/group/SalesStaff?includeIndirectMappings=true
```

In this case 'group' is one of the properties of a 'userToGroupMapping' resource/element, so it made sense to include that in the URL (as a primary filter criteria... in this case filtering for a group called 'SalesStaff').  'includeIndirectMappings' by contrast is not a property of a 'userToGroupMapping', but rather it specifies how the data should be searched (moreso a secondary filter criteria), and hence it's included in the query string.

There are several different approaches here, but what's important is choosing the one which best fits your use-case, and moreso **applying that choice consistently across all endpoints in your REST API**.

#### URL Sections/Regions

The ApplicationAccess 'groupToEntityMappings' GET endpoints discussed above bring up another important point... if you decide to use the URL to define filters and/or other resource properties, I find it helps to break the URL down into sections, and then adhere to the same sectioning pattern across endpoints.  In ApplicationAccess, URLs tend to be broken into the following 4 sections...

```
http://127.0.0.1:5000/api/v1/groupToEntityMappings/entityType/{entityType}/entity/{entity}
                      |   |  |                     |
        'api' prefix  ^   |  |                     |
                          |  |                     |
              API version ^  |                     |
                             |                     |
                             ^ Resource            |
                                                   |
                                                   ^ Filtering of the resource
```

#### Non-CRUD Operations

As mentioned above, in an application that does anything over and above serving up data via CRUD, you'll have functions in that application that don't fit the available HTTP methods... functions like 'convert', 'process', 'validate', etc...  Broadly there are two approaches I would use to implement these...

**Change your function verb into a noun**

Rather than performing a validate() function, you could instead get() a 'validation' resource, using a URL like...

| URL Path | Method | Description |
| -------- | ------ | ----------- |
| /validation/groupToEntityMappings/group/{group}/entityType/{entityType}/entity/{entity} | GET | Validates the specified group to entity mapping. |

**Use a scheme to represent the non-HTTP method**

Google's API documentation suggests a nice scheme for [custom methods](https://google.aip.dev/136), using a POST method with a colon in the URL followed by the non-standard method/verb.  For example to sort() books...

```
/v1/{parent=publishers/*}/books:sort
```

ApplicationAccess adopts this scheme for administrator functions in some components, e.g...

| URL Path | Method | Description |
| -------- | ------ | ----------- |
| /operationProcessing:pause | POST | Pauses/holds any incoming operation requests. |
| /operationProcessing:resume | POST | Resumes any incoming operation requests following a preceding pause. |

One thing I would be careful _NOT_ to do, would be to put non-standard verbs in your URLs and break any section/region pattern you'd used for CRUD methods.  For example if you had CRUD endpoints divided into sections (as [described above](#url-sectionsregions))...

```
http://127.0.0.1:5000/api/v1/groupToEntityMappings/entityType/{entityType}/entity/{entity}
```

...then having non-CRUD endpoints which look the same as CRUD endpoints (but actually represent non-HTTP functions) e.g....

```
http://127.0.0.1:5000/api/v1/operationProcessing/pause
```

...would create inconsistency and make your API harder to understand and consume.  At a glance, based on patterns in the CRUD endpoints, I would assume a POST endpoint with the above URL might be used to create() a resource/entity representing a pause in operation processing, rather than to actually pause() operation processing.


### PUT vs POST vs PATCH for Inserts and Updates

According to the [MDN HTTP reference](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference), PUT, POST, and PATCH methods have the following use cases and [idemoptency](https://developer.mozilla.org/en-US/docs/Glossary/Idempotent)...

| Method | Use Case | Idemopotency |
| ------ | -------- | ------------ |
| PUT | The PUT method replaces all current representations of the target resource with the request content. | Yes |
| POST | The POST method submits an entity to the specified resource, often causing a change in state or side effects on the server. | No |
| PATCH | The PATCH method applies partial modifications to a resource. | No |

Hence, for 'create' and 'update' CRUD functions you can pick the HTTP method which best matches your use case.   'Create' functions in the standard version of ApplicationAccess are not idempotent (an error will result if the specified element already exists), hence I chose to use POST methods.  The dependency-free version of ApplicationAccess does implement idempotent 'create's, but I still chose to implement these as POST methods rather than (the arguably more appropriate) PUT methods... in this case I chose to preserve API consistency between ApplicationAccess versions, over adhering 100% to REST method guidelines.

### HTTP Status Codes

HTTP defines a fairly rich set of [response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status), so you should utilize these to ensure that your API is returning as much information as possible to clients in cases of error.  Developers often fall into the lazy habit of not catching application exceptions, and leave this to the web hosting framework, which usually results in all errors being returned with [500 status](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/500)... _don't_ do this!  By catching exceptions and returning appropriate HTTP status codes, it allows consumers of your API make better decisions on how to act when something goes wrong.

In ApplicationAccess, custom middleware is used to catch exceptions and automatically map to and return appropriate HTTP status codes.  This is implemented via the [ExceptionToHttpStatusCodeConverter](https://github.com/alastairwyse/ApplicationAccess/blob/58c28d81cd8be9deb506e5be33dabb8df7adc2ff/ApplicationAccess.Hosting.Rest.Utilities/ExceptionToHttpStatusCodeConverter.cs) class, and [MiddlewareUtilities.SetupExceptionHandler()](https://github.com/alastairwyse/ApplicationAccess/blob/58c28d81cd8be9deb506e5be33dabb8df7adc2ff/ApplicationAccess.Hosting.Rest/MiddlewareUtilities.cs#L60) method.  The benefit of implementing this mapping through common middleware is that it avoids repeated try/catch statements throughout all controller methods which perform the same catch/map logic.  The ExceptionToHttpStatusCodeConverter class can be configured to include customed mappings, but supports the following by default...

| .NET Exception | HTTP Status Code | 
| -------------- | ---------------- |
| [ArgumentException](https://learn.microsoft.com/en-us/dotnet/api/system.argumentexception?view=net-10.0) | [400 Bad Request](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/400) |
| [NotFoundException](https://github.com/alastairwyse/ApplicationAccess/blob/58c28d81cd8be9deb506e5be33dabb8df7adc2ff/ApplicationAccess.Hosting.Rest.Utilities/NotFoundException.cs) | [404 Not Found](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/404) |
| [Exception](https://learn.microsoft.com/en-us/dotnet/api/system.exception?view=net-10.0) | [500 Internal Server Error](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/500) |

ExceptionToHttpStatusCodeConverter also recognises inheritance hierarchies, so that (for example) exceptions which derive from ArgumentException ([ArgumentNullException](https://learn.microsoft.com/en-us/dotnet/api/system.argumentnullexception?view=net-10.0), [ArgumentOutOfRangeException](https://learn.microsoft.com/en-us/dotnet/api/system.argumentoutofrangeexception?view=net-10.0), etc...) are also mapped to 400 status codes.

I implemented equivalent exception to status code mapping functionality in Spring Boot in class [GlobalControllerExceptionHandler](https://github.com/alastairwyse/JavaTaskManager/blob/27a9e01af8e29082ad8c030910fa8a2814446b95/api/src/main/java/net/alastairwyse/taskmanager/api/GlobalControllerExceptionHandler.java).

#### Return 405 Status for Invalid HTTP Methods

Your API should return a [405 Method Not Allowed](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/405) if it receives a request for a URL with a HTTP method which is not supported for that URL.

### Exceptions

REST itself doesn't specify a format for the response body in the case of errors.  However in addition to returning an appropriate HTTP status code, the response body should contain sufficient imformation for the client to understand the error, and take action accordingly.  I'm aware of 2 published standards for JSON error responses, the [Microsoft REST API Guideilnes](https://github.com/microsoft/api-guidelines/blob/vNext/graph/Guidelines-deprecated.md#7102-error-condition-responses), and [RFC 9457 Problem Details](https://datatracker.ietf.org/doc/html/rfc9457).  My thoughts/opinion on these two standards are below...

#### RFC 9457 Problem Details
It's plus that it's a standard (superseding the original [RFC 7807 Problem Details](https://datatracker.ietf.org/doc/html/rfc7807)) that has been around for some time, and is implemented either by default or as a configuration option in many API frameworks.  It's prevalence means that adopting it gives a good chance that clients will be able to easy consume the JSON format it defines.  The thing I don't like is that it doesn't have any built-in mechanism for handling exceptions hierarchies (i.e. inner or causing errors).  Potentially this doesn't matter too much if your API is consumed directly by a UI/front-end, as in this case errors are likely to be just logged and/or reported to the user.  But in situations were your API is consumed programmatically by other systems, not being able to pass the whole hierarchy of an error to clients is a significant limitation.  Java Exceptions have a [getCause()](https://docs.oracle.com/javase/8/docs/api/java/lang/Throwable.html#getCause--) method, .NET Exceptions have an [InnerException property](https://learn.microsoft.com/en-us/dotnet/api/system.exception.innerexception?view=net-10.0), Python Exceptions have a [\_\_cause\_\_ attribute](https://docs.python.org/3/library/exceptions.html#BaseException.__cause__)... it's a shame the RFC 9457 standard doesn't define a way to include these.

#### Microsoft REST API Guideilnes
I like this more than RFC 9457 as it does define an 'innererror' property.  I've always been a bit confused as to why they define [Error](https://github.com/microsoft/api-guidelines/blob/vNext/graph/Guidelines-deprecated.md#error--object) and [InnerError](https://github.com/microsoft/api-guidelines/blob/vNext/graph/Guidelines-deprecated.md#innererror--object) as different types... wouldn't it have been easier to just merge them into the same type?... this would match the Exception object's structure in major languages (e.g. Java's Exception.getCause() returns a [Throwable](https://docs.oracle.com/javase/8/docs/api/java/lang/Throwable.html), .NET's Exception.InnerException returns another [Exception](https://learn.microsoft.com/en-us/dotnet/api/system.exception?view=net-10.0), Python's Exception.\_\_cause\_\_ should contain another Python [Exception](https://docs.python.org/3/library/exceptions.html#Exception)).  However this aside, the Microsoft format is not a bad choice.

In ApplicationAccess I took inspiration from the Microsoft format, but 'corrected' the distinction between the Error and InnerError described above.  Error objects have a the following properties...

| Property Name | Data Type | Description |
| ------------- | --------- | ----------- |
| code | String | An internal code representing the error.  In ApplicationAccess this is populated with the class name of the exception which the error represents. |
| message | String | A description of the error. |
| target | String | The target of the error.  In ApplicationAccess this is populated with the [TargetSite property](https://learn.microsoft.com/en-us/dotnet/api/system.exception.targetsite?view=net-10.0#system-exception-targetsite) of the exception which the error represents. |
| attributes | Array of Object | A collection of name/value pairs which give additional details of the error. |
| innererror | Object (with the same error object structure) | The error which caused this error. |

For example, attempting to add a user who already exists results in a 400 (bad request) status, and the following JSON response body...

```json
{
  "error": {
    "code": "ArgumentException",
    "message": "User 'John' already exists. (Parameter 'user')",
    "target": "AddUser",
    "attributes": [
      {
        "name": "ParameterName",
        "value": "user"
      }
    ],
    "innererror": {
      "code": "LeafVertexAlreadyExistsException`1",
      "message": "Vertex 'John' already exists in the graph.",
      "target": "AddLeafVertex"
    }
  }
}
```

Trying to retrieve a user who doesn't exist results in a 404 (not found) status, and...

```json
{
  "error": {
    "code": "NotFoundException",
    "message": "User 'Paul' does not exist.",
    "target": "ContainsUser",
    "attributes": [
      {
        "name": "ResourceId",
        "value": "Paul"
      }
    ]
  }
}
```

#### Hiding Details

If your REST API is exposed over public internet, you may want to consider either hiding inner details of exceptions (as complete exception hierarchies might expose implementation details of your system to malicious actors), or masking the externally exposed errors with some generic message (whilst likely logging/storing the full details internally).  ApplicationAccess allows options to both replace the exception message with a specified generic message, and to remove any inner exceptions from the returned error object.

#### Mapping Exceptions to JSON

In a similar way to how exceptions are automatically mapped to HTTP status codes ([described above](#http-status-codes)), ApplicationAccess automatically maps and converts exceptions to its standard JSON error format via ASP.NET middleware.  Class [ExceptionToHttpErrorResponseConverter](https://github.com/alastairwyse/ApplicationAccess/blob/58c28d81cd8be9deb506e5be33dabb8df7adc2ff/ApplicationAccess.Hosting.Rest.Utilities/ExceptionToHttpErrorResponseConverter.cs) and method and [MiddlewareUtilities.SetupExceptionHandler()](https://github.com/alastairwyse/ApplicationAccess/blob/58c28d81cd8be9deb506e5be33dabb8df7adc2ff/ApplicationAccess.Hosting.Rest/MiddlewareUtilities.cs#L60) implement this mapping and conversion.  Class [GlobalControllerExceptionHandler](https://github.com/alastairwyse/JavaTaskManager/blob/27a9e01af8e29082ad8c030910fa8a2814446b95/api/src/main/java/net/alastairwyse/taskmanager/api/GlobalControllerExceptionHandler.java) performs similar functionality in Java / Spring Boot.

### MIME Types, 'Accept' and 'Content-Type' Headers

#### Utilizing the 'Accept' Header

The 'accept' header in an HTTP request is used by a client to tell your API which content/data types it can support.  It can also be used to allow API endpoints to return multiple data types (i.e. aside from JSON).  As an example, rather than creating an endpoint to return a report in PDF format like this...

| URL Path | Method | Description |
| -------- | ------ | ----------- |
| reports/date/{date}:convertToPdf | GET | Get a report for the specified date as a PDF document |

... you could instead omit the ':convertToPdf' from the URL, and when the request contains a header like this...

```
accept: application/pdf
```

...return the report as a PDF document.

As a general rule, if you need to return non-JSON data from your REST API, and the data type is a recognized [MIME/media type](https://www.iana.org/assignments/media-types/media-types.xhtml), you should consider utilizing the 'accept' header to define/differentiate the data type.

#### Reject Requests for Unsupported MIME/media Types

When creating REST APIs which only support JSON, it's easy for developers to ignore the 'accept' header, and just blindly return JSON regardless of what it contains.  This isn't good practice... if I client requests a specific data/content type, they expect to either receive that data/content type in the response, or a [406 Not Acceptable](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/406) error status.  This is simple enough to implement in ASP.NET by firstly annotating your controller methods with the [Produces attribute](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.producesattribute?view=aspnetcore-10.0), e.g...

```c#
/// <summary>
/// Returns all groups.
/// </summary>
/// <returns>All groups.</returns>
[HttpGet]
[Route("groups")]
[Produces(MediaTypeNames.Application.Json)]
public IEnumerable<String> Groups()
{
    return groupQueryProcessor.Groups;
}
```

...and then during setup, setting the [RespectBrowserAcceptHeader](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.mvcoptions.respectbrowseracceptheader?view=aspnetcore-10.0) and [ReturnHttpNotAcceptable](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.mvcoptions.returnhttpnotacceptable?view=aspnetcore-10.0) options true...

```c#
builder.AddMvcOptions(options =>
{
    options.RespectBrowserAcceptHeader = true;
    options.ReturnHttpNotAcceptable = true;
});
```

There may be a similar way to configure similar behaviour in Spring Boot.  I was unable it find it, but equivalent functionality can be implemented as a custom interceptor as [demonstrated here](https://github.com/alastairwyse/JavaTaskManager/blob/27a9e01af8e29082ad8c030910fa8a2814446b95/api/src/main/java/net/alastairwyse/taskmanager/api/AcceptHeaderParsingHandlerInterceptor.java).

#### 'Content-Type' Headers

TODO!!

### Swagger Doco

Clearly documenting your API endpoints through Swagger is critical.  Don't leave your consumers guessing/assuming the behaviour of an API endpoint from its URL... clearly document the endpoint functionality and parameters.  This can be done easily in ASP.NET and Spring Boot by either XML commenting the controller methods (C#)...

```c#
/// <summary>
/// Returns all groups.
/// </summary>
/// <returns>All groups.</returns>
[HttpGet]
[Route("groups")]
[Produces(MediaTypeNames.Application.Json)]
public IEnumerable<String> Groups()
{
    return groupQueryProcessor.Groups;
}
```

...or using the [Operation annotation](https://docs.swagger.io/swagger-core/v2.0.0-RC3/apidocs/io/swagger/v3/oas/annotations/Operation.html) (Spring Boot)...

```java
@Operation(summary = "Updates a task")
@PutMapping("")
@ApiResponse(responseCode = "200", description = "Task updated successfully")
@ApiResponse(responseCode = "404", description = "A task with the specified id does not exist", content = @Content)
public ResponseEntity<?> updateTask(@RequestBody TaskDto taskDto) throws TaskDoesntExistException {

    var task = new Task(taskDto);
    taskManager.updateTask(task);

    return new ResponseEntity<Void>(HttpStatus.OK);
}
```

### Versioning

Even if you don't plan to utilize versioned endpoints in your API, you should include the ability to version as a default.  If you don't enable versioning from the outset, and then find you need to introduce it, you'll likely have to either...

 * Introduce breaking changes to support versioning, or...
 * Implement versioning in a non-standard way to avoid introducing breaking changes

Regarding how to implement versioning, Troy Hunt wrote a [great article on this](https://www.troyhunt.com/your-api-versioning-is-wrong-which-is/) some years ago which is worth reading.  There are basically 3 common ways you can implement versioning in a REST API...

 1. Include the version in the URL (e.g. as I do in the [ApplicationAccess examples above](#url-sectionsregions))
 2. Use a custom HTTP header in the request (e.g. 'api-version: 1')
 3. Include the version in the querystring (e.g. 'http://127.0.0.1:5000/api/groups?version=1')

The aforementioned article actually discusses a 4th method which is to use a custom content type.  My opinion on each of these methods is as follows...

**URL** - This is my preferred way of versioning.  As outlined above, I like the idea of breaking the URL into [distinct sections](#url-sectionsregions).  I also feel that the API version should be treated as a key / first class property of the data/function definition, so it makes sense to me to have a permanent (and consistent) place to define the version at the start of all endpoint URLs.

**Custom HTTP header** - This would be my second choice.  It makes the version a little less 'visible' to human observers of the API surface, but given there are other mandatory properties in the request header (e.g. 'accept') it feels that this method supports treating the API version as a key / first class property (as described above).

**Query String** - I don't like this method as much, as I feel that query strings are best used for optional properties of the data/function definition (as [discussed above](#data-identifying-and-filtering)).  However you may find a compelling reason to implement versioning this way in your API, and again as long as it's applied consistently, that's fine.

**Custom Content Type** - This would be my last choice.  There is already a defined standard of [MIME/media type values](https://www.iana.org/assignments/media-types/media-types.xhtml), and although the standard offers scope for your own custom values (via 'vnd', and 'prs' prefixes), it just feels like you might cause more difficultly for clients trying to consume your API, if for example they're using a HTTP package which doesn't have the provision to support custom MIME/media types.  Still if this is the right implementation for your use case, as always, it should just be applied consistenly across your API surface.




