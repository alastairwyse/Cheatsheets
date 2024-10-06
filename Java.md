#### Implementing AutoCloseable (equivalent of C# IDisposable)

In class definition...

```java
public abstract class AccessManagerClientBase<TUser, TGroup, TComponent, TAccess> implements AutoCloseable {
        
    /** The client to use to connect. */
    protected HttpClient httpClient;
}
```

close() method

```java
//#region Close Method

@Override
public void close() throws IOException {
    if (httpClientInstantiatedInConstructor == true) {
        httpClient.close();
    }
}

//#endregion
```

#### 'try with resources' (equivalent of C# 'using')

```java
try(BufferedReader inputReader = new BufferedReader(new InputStreamReader(System.in));
    FileApplicationLogger remoteReceiverLog = new FileApplicationLogger(LogLevel.Debug, '|', "  ", "C:\\Temp\\JavaReceiver.log");
    TcpRemoteReceiver tcpReceiver = new TcpRemoteReceiver(55000, 10, 1000, 25, 1024, remoteReceiverLog);
    TcpRemoteSender tcpSender = new TcpRemoteSender(InetAddress.getLoopbackAddress(), 55001, 10, 1000, 30000, 25, remoteReceiverLog)) {
    
    // Do stuff with above objects
    // close() is called automatically at the end of the code block
}
```

#### Inner classes

Just declare within the body of another class... similar to C#...

```java
    //#region Nested Classes

    /**
     * Container/model class holding parameters passed to a routine which handles a {@link HttpResponse}.
     */
    protected class ResponseFunctionParameters {

        /** The HTTP method called in the request which creatd the response. */
        protected HttpMethod httpMethod;
        
        // etc...
    }

    //#endregion
```

#### String equality

Have to use the equals() method when comparing strings.  '==' operator is not guaranteed to return true for identical strings...

```java
if (httpErrorResponse.getCode().equals("ArgumentNullException")) {
    throw new NullPointerException(httpErrorResponse.getMessage());
}
```

#### Generic method declaration

Slightly different syntax to C#...

```java
protected <T> T getIterableElement(Iterable<T> iterable, int index) {

    // etc...
}
```

#### String value substitution

Use String.format()...

```java
String.format(
    "Failed to call URL '%s' with '%s' method.  Error deserializing response body from JSON to type.", 
    requestUrl.toString(), 
    HttpMethod.GET
)
```

#### Exception equivalent to C#

| C# Exception | Java Error/Exception | Notes |
| ------------ | -------------------- | ----- |
| ArgumentException | IllegalArgumentException | Includes C# exceptions derived from ArgumentException... ArgumentOutOfRangeException, etc... |
| ArgumentNullException | NullPointerException | Many conflicting opinions on this... although apparently [many JDK methods use NullPointerException for null parameters](https://www.baeldung.com/java-illegalargumentexception-or-nullpointerexception#2-its-consistent-with-jdk-apis).  Likely choose to use or not and then ensure it's consistent throughout a solution. |

#### Deriving from Exception

Can also extend RuntimeException

```java
/**
 * The exception that is thrown when deserialization fails.
 */
public class DeserializationException extends Exception {
    
    /**
     * Constructs a DeserializationException.
     * 
     * @param message The detail message. The detail message is saved for later retrieval by the Throwable.getMessage() method.
     */
    public DeserializationException(String message) {
        super(message);
    }
    
    /**
     * Constructs a DeserializationException.
     * 
     * @param message The detail message. The detail message is saved for later retrieval by the Throwable.getMessage() method.
     * @param cause The cause (which is saved for later retrieval by the Throwable.getCause() method). (A null value is permitted, and indicates that the cause is nonexistent or unknown.)
     */
    public DeserializationException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

#### Exception testing in unit tests

```java
TaskDoesntExistException e = assertThrows(TaskDoesntExistException.class, () -> 
{
    testDefaultTaskManager.deleteTask(testTask);
});

assertTrue(e.getMessage().contains(String.format("A task with id '%s' does not exist in the task manager.", testTask.getId())));
```

#### Inheriting base method documentation and decorating with implementation-specific method details 

```java
/**
 * @inheritDoc
 * @exception RuntimeException If a non-success response status was received.
 * @exception RuntimeException If the response could not be deserialized to an object.
 * @exception IOException If an I/O error occurs when sending or receiving, or the client has shut down.
 * @exception InterruptedException If the operation is interrupted.
 */
@Override
public List<TUser> getUsers() throws IOException, InterruptedException {

    // etc...
}
```    

#### Jackson 'JsonAlias' annotation

Used to override the JSON name of properties auto-serialized using Jackson (e.g. for DTO classes)...

```java
/**
 * DTO container class holding an application component, and level of access to that component.
 */
public class ApplicationComponentAndAccessLevel {
    
    @JsonAlias({ "applicationComponent" })
    public String ApplicationComponent;

    @JsonAlias({ "accessLevel" })
    public String AccessLevel;
}
```

#### Overriding the equals() and hashCode() methods for model/container classes...

Note it's not generic like C#... hence extra type checks required

```java
@Override
public boolean equals(Object other) {
    if (this == other) {
        return true;
    }
    if (other == null) {
        return false;
    }
    if (this.getClass() != other.getClass()) {
        return false;
    }
    ApplicationComponentAndAccessLevel<TComponent, TAccess> typedOther = (ApplicationComponentAndAccessLevel<TComponent, TAccess>)other;

    return (this.applicationComponent.equals(typedOther.applicationComponent) && this.accessLevel.equals(typedOther.accessLevel));
}

@Override
public int hashCode() {
    return (this.applicationComponent.hashCode() * prime1 + this.accessLevel.hashCode() * prime2);
}
```

#### Implementing hashCode() for model/container classes with multiple properties...

Can use [Objects.hash()](https://docs.oracle.com/javase/8/docs/api/java/util/Objects.html#hash-java.lang.Object...-).  Seems to be similar to C# [HashCode.Combine()](https://learn.microsoft.com/en-us/dotnet/api/system.hashcode.combine?view=net-8.0).


#### Springboot

See [JavaTaskManager](https://github.com/alastairwyse/JavaTaskManager?tab=readme-ov-file#spring-boot-and-aspnet-core-comparison) for a comparison between Springboot and ASP.NET Core.

