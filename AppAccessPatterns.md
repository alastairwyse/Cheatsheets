AppAccess Options Pattern
-------------------------

Some more complex options like DatabaseConnectionOptions have an associated Parser class.
The pattern is that these are used to parse the *Options and convert to a more standard/structured container class or record... like DatabaseConnectionParameters as an example
Then this more structured parameters class is passed to a factory to make a concrete instance of some 'doing' class 
Then the 'ConfigureOptionsAction' on the AppInitializer object configures what validation to run (defined in Program.cs)
  Custom validators can be called in here (e.g. one described above), aswell as validation by standard annotations (via .ValidateDataAnnotations().ValidateOnStart())

Hmmm... but we also have MetricLoggingOptionsValidator
  Which is called from ApplicationInitializer
  This just validates... e.g. in the case of metrics logging we don't have two logging destinations defined
  The Validate() method is void
  There's also a DatabaseConnectionOptionsValidator, so there's a Parser and a Validator for DB connection options

EventCacheConnectionOptions
  Will need an enum to specify gRPC or REST
  2x 'Retry' properties will need to be made optional, and then will likely need a parser class
    and convert to a Parameters version of the class
    and then that needs to be passed to a factory
    and factory should instantiate an instance of client
  Currently Reader/Writer hosted service nodes create a fixed instance of REST EventCacheClient, but will need to call a factory method
  
Exception Handling Middleware
-----------------------------

Just to refresh on this we basically have ability to define exception handling/conversion at 3 levels...

  1. Middleware/Interceptor -> since we want this functionality to be portable to other projects, this should only include C# standard exceptions.
      ** BUT exists in ApplicationAccess.Hosting.Rest.Utilities namespace, and NotFoundException is also here... so maybe no problem with NotFoundException 
  2. Application initializer -> this should be for things that are non configurable, and are standard to all AppAccess components (e.g. UserNotFoundException etc...)
  3. Program.cs -> things that are either user configurable (e.g. TripswitchException), or custom to a specific AppAccess component.