### Lessons Learned from MongoDB Implementation in [ApplicationAccess](https://github.com/alastairwyse/ApplicationAccess)

#### Integration Testing with EphemeralMongo

When writing automated tests for external dependencies (databases, external APIs, etc...), I usually prefer to write unit tests against interfaces to those external dependencies (utilizing mocks of the interfaces) rather than instantiating actual instances of the external dependencies and testing against those.  The reasoning for this is...

1. Testing against mocks purely within memory is much more performant than instantiating and stepping out into external code/components/platforms.
2. By instantiating external code/components/platforms, you're often testing those components in addition to your own, and that shouldn't be a concern of your tests (those components should have their own test suites).

If you step outside the memory space of the code under test with I/O (disk, network, etc...), the tests then fall more into the category of integration tests.  That said integration tests are often a useful and/or necessary supplement to unit tests, but as a principle I like to have unit tests covering my code as a baseline.

With this principle in mind, in previous MongoDB-based projects I wrote unit tests for the MongoDB interfacing code by mocking the interfaces to MongoDB ([IMongoClient](https://mongodb.github.io/mongo-csharp-driver/3.5.0/api/MongoDB.Driver/MongoDB.Driver.IMongoClient.html), [IMongoDatabase](https://mongodb.github.io/mongo-csharp-driver/3.5.0/api/MongoDB.Driver/MongoDB.Driver.IMongoDatabase.html), [IMongoCollection&lt;T&gt;](https://mongodb.github.io/mongo-csharp-driver/3.5.0/api/MongoDB.Driver/MongoDB.Driver.IMongoCollection-1.html)), and by shimming and mocking searching and filtering extension methods (like [IMongoCollection&lt;T&gt;.Find()](https://mongodb.github.io/mongo-csharp-driver/3.5.0/api/MongoDB.Driver/MongoDB.Driver.IMongoCollectionExtensions.Find.html)).  In hindsight this was a somewhat arduous exercise... quite a few mock responses need to be defined even to do a basic read from MongoDB, but it was ultimately worthwhile in those projects as the MongoDB interfacing code was simple... usually just requiring simple reads/writes of single documents (querying only by unique key).

ApplicationAccess required more complex MongoDB interfacing code than those previous projects.  The code extracts from the [MongoDbAccessManagerTemporalBulkPersister](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Persistence.MongoDb/MongoDbAccessManagerTemporalBulkPersister.cs) class below implement removing a user to entity mapping...

```C#
protected void RemoveUserToEntityMapping(IClientSessionHandle session, TUser user, String entityType, String entity, Guid eventId, DateTime transactionTime)
{
    String stringifiedUser = userStringifier.ToString(user);
    IMongoCollection<UserToEntityMappingDocument> userToEntityMappingCollection = database.GetCollection<UserToEntityMappingDocument>(userToEntityMappingsCollectionName);
    FilterDefinition<UserToEntityMappingDocument> existingUserToEntityMappingFilter = AddTemporalTimestampFilter(transactionTime, Builders<UserToEntityMappingDocument>.Filter.And
    (
        Builders<UserToEntityMappingDocument>.Filter.Eq(document => document.User, stringifiedUser),
        Builders<UserToEntityMappingDocument>.Filter.Eq(document => document.EntityType, entityType),
        Builders<UserToEntityMappingDocument>.Filter.Eq(document => document.Entity, entity)
    ));
    GetExistingDocument
    (
        session,
        userToEntityMappingCollection,
        existingUserToEntityMappingFilter,
        transactionTime,
        GenerateRemoveElementFindExistingDocumentFailedExceptionMessage(userToEntityMappingsCollectionName, "user to entity mapping"),
        GenerateRemoveElementNoDocumentExistsExceptionMessage(transactionTime, Tuple.Create("user", stringifiedUser), Tuple.Create("entity type", entityType), Tuple.Create("entity", entity))
    );

    // Invalidate the user to entity mapping
    UpdateDefinition<UserToEntityMappingDocument> invalidationUpdate = Builders<UserToEntityMappingDocument>.Update.Set(document => document.TransactionTo, SubtractTemporalMinimumTimeUnit(transactionTime));
    try
    {
        UpdateOne(session, userToEntityMappingCollection, existingUserToEntityMappingFilter, invalidationUpdate);
    }
    catch (Exception e)
    {
        throw new Exception($"Failed to invalidate user to entity mapping document in collecion '{userToEntityMappingsCollectionName}' when removing user to entity mapping from MongoDB.", e);
    }
    CreateEvent(session, eventId, transactionTime);
}

protected FilterDefinition<T> AddTemporalTimestampFilter<T>(DateTime transactionTime, params FilterDefinition<T>[] existingFilters)
    where T : DocumentBase
{
    IEnumerable<FilterDefinition<T>> allFilters = new List<FilterDefinition<T>>(existingFilters)
        .Append(Builders<T>.Filter.Lte(document => document.TransactionFrom, transactionTime))
        .Append(Builders<T>.Filter.Gte(document => document.TransactionTo, transactionTime));

    return Builders<T>.Filter.And(allFilters);
}

protected T GetExistingDocument<T>
(
    IClientSessionHandle session,
    IMongoCollection<T> collection,
    FilterDefinition<T> baseFilterDefinition,
    DateTime transactionTime,
    String findFailedExceptionDescription,
    String noDocumentExistsExceptionDescription
)
    where T : DocumentBase
{
    FilterDefinition<T> filterDefinition = AddTemporalTimestampFilter(transactionTime, baseFilterDefinition);
    T existingDocument;
    try
    {
        existingDocument = Find(session, collection, filterDefinition).FirstOrDefault();
    }
    catch (Exception e)
    {
        throw new Exception(findFailedExceptionDescription, e);
    }
    if (existingDocument == null)
        throw new Exception(noDocumentExistsExceptionDescription);

    return existingDocument;
}

protected UpdateResult UpdateOne<T>(IClientSessionHandle? session, IMongoCollection<T> collection, FilterDefinition<T> filterDefinition, UpdateDefinition<T> updateDefinition)
{
    if (session == null)
    {
        return collection.UpdateOne(filterDefinition, updateDefinition);
    }
    else
    {
        return collection.UpdateOne(session, filterDefinition, updateDefinition);
    }
}

protected void CreateEvent(IClientSessionHandle session, Guid eventId, DateTime transactionTime)
{
    IMongoCollection<EventIdToTransactionTimeMappingDocument> eventIdToTransactionTimeMappingCollection = database.GetCollection<EventIdToTransactionTimeMappingDocument>(eventIdToTransactionTimeMapCollectionName);
    DateTime lastTransactionTime = DateTime.MinValue;
    Int32 lastTransactionSequence = 0;
    Int32 transactionSequence = 0;

    // Get the most recent transaction time
    try
    {
        EventIdToTransactionTimeMappingDocument lastTransactionTimeDocument = Find(session, eventIdToTransactionTimeMappingCollection, FilterDefinition<EventIdToTransactionTimeMappingDocument>.Empty)
            .SortByDescending(document => document.TransactionTime)
            .FirstOrDefault();
        if (lastTransactionTimeDocument != null)
        {
            // Get the largest transaction sequence within the most recent transaction time
            FilterDefinition<EventIdToTransactionTimeMappingDocument> lastTransactionTimeFilter = Builders<EventIdToTransactionTimeMappingDocument>.Filter.Eq(document => document.TransactionTime, lastTransactionTimeDocument.TransactionTime);
            EventIdToTransactionTimeMappingDocument lastTransactionSequenceDocument = Find(session, eventIdToTransactionTimeMappingCollection, lastTransactionTimeFilter)
                .SortByDescending(document => document.TransactionSequence)
                .FirstOrDefault();
            lastTransactionTime = lastTransactionSequenceDocument.TransactionTime;
            lastTransactionSequence = lastTransactionSequenceDocument.TransactionSequence;
        }
    }
    catch (Exception e)
    {
        throw new Exception($"Failed to retrieve most recent transaction time and sequence from collection '{eventIdToTransactionTimeMapCollectionName}' when persisting a new event to MongoDB.", e);
    }

    if (transactionTime < lastTransactionTime)
    {
        throw new ArgumentException($"Parameter '{nameof(transactionTime)}' with value '{transactionTime.ToString(dateTimeExceptionMessageFormatString)}' must be greater than or equal to last transaction time '{lastTransactionTime.ToString(dateTimeExceptionMessageFormatString)}'.", nameof(transactionTime));
    }
    else if (transactionTime == lastTransactionTime)
    {
        transactionSequence = lastTransactionSequence + 1;
    }

    // Create the new event
    EventIdToTransactionTimeMappingDocument newDocument = new()
    {
        EventId = eventId,
        TransactionTime = transactionTime,
        TransactionSequence = transactionSequence
    };
    try
    {
        InsertOne(session, eventIdToTransactionTimeMappingCollection, newDocument);
    }
    catch (Exception e)
    {
        throw new Exception($"Failed to insert document into collection '{eventIdToTransactionTimeMapCollectionName}'.", e);
    }
}

protected IFindFluent<T, T> Find<T>(IClientSessionHandle? session, IMongoCollection<T> collection, FilterDefinition<T> filter)
{
    if (session == null)
    {
        return collection.Find(filter);
    }
    else
    {
        return collection.Find(session, filter);
    }
}

protected void InsertOne<T>(IClientSessionHandle? session, IMongoCollection<T> collection, T document)
{
    if (session == null)
    {
        collection.InsertOne(document);
    }
    else
    {
        collection.InsertOne(session, document);
    }
}
```

To success-test RemoveUserToEntityMapping() with purely interface mocks would require setting ~7 mock call expectations and associated return data.  Plus, to really test thoroughly you should be confirming the values of parameters passed to the mocks, and this gets tricky with things like [FilterDefinition&lt;T&gt;](https://mongodb.github.io/mongo-csharp-driver/3.5.0/api/MongoDB.Driver/MongoDB.Driver.FilterDefinition-1.html) objects which (based on my earlier experience) don't have value-based equality compare methods and require techniques like comparing serialized versions in order to check that expected and actual parameter values match.  Around 3 such filter parameter checks would be required to properly test this method, and when you consider that there are more complex MongoDB interfacing methods in this class, it becomes clear that mock-based unit tests are going to be quite complex and a burden to maintain.

Fortunately there's a better way to ensure sufficient test coverage of your MongoDB interfacing code... [EphemeralMongo](https://github.com/asimmon/ephemeral-mongo) is a package which seamlessly runs a real temporary/disposable MongoDB database behind your tests.  No external processes or scripts are required... it can be instantiated with just simple C# code in the 'setup' routines of your test class, like below...

```C#
[OneTimeSetUp]
protected void OneTimeSetUp()
{
    var mongoRunnerOptions = new MongoRunnerOptions();
    mongoRunnerOptions.UseSingleNodeReplicaSet = true;
    mongoRunner = MongoRunner.Run(mongoRunnerOptions);
    mongoClient = new MongoClient(mongoRunner.ConnectionString);
    mongoDatabase = mongoClient.GetDatabase("ApplicationAccess");
}
```

This approach does fall into the category of what I consider integration tests rather than unit tests as discussed above... the suite of MongoDB-related tests complete in seconds rather than milliseconds... but for the simplification of test code that EphemeralMongo brings, the trade off is well worth it.  Using EphemeralMongo, the success tests for the RemoveUserToEntityMapping() method discussed above is covered in a reasonable number of lines of code, and is far more readable and maintainable than the mock-based equivalent...

```C#
[Test]
public void RemoveUserToEntityMapping()
{
    List<UserToEntityMappingDocument> initialUserToEntityMappingDocumentDocuments = new()
    {
        new UserToEntityMappingDocument() {  User = "user1", EntityType = "ClientAccount", Entity = "CompanyA", TransactionFrom = CreateDataTimeFromString("2025-10-04 17:33:52.0000000"), TransactionTo = CreateDataTimeFromString("2025-10-09 07:53:28.0000000")},
        new UserToEntityMappingDocument() {  User = "user2", EntityType = "ClientAccount", Entity = "CompanyA", TransactionFrom = CreateDataTimeFromString("2025-10-04 17:33:53.0000000"), TransactionTo = DateTime.MaxValue},
        new UserToEntityMappingDocument() {  User = "user1", EntityType = "Suppliers", Entity = "CompanyA", TransactionFrom = CreateDataTimeFromString("2025-10-04 17:33:53.0000000"), TransactionTo = DateTime.MaxValue},
        new UserToEntityMappingDocument() {  User = "user1", EntityType = "ClientAccount", Entity = "CompanyB", TransactionFrom = CreateDataTimeFromString("2025-10-04 17:33:53.0000000"), TransactionTo = DateTime.MaxValue},
        new UserToEntityMappingDocument() {  User = "user1", EntityType = "ClientAccount", Entity = "CompanyA", TransactionFrom = CreateDataTimeFromString("2025-10-09 08:26:41.0000000"), TransactionTo = DateTime.MaxValue}
    };
    userToEntityMappingsCollection.InsertMany(initialUserToEntityMappingDocumentDocuments);
    String user = "user1";
    String entityType = "ClientAccount";
    String entity = "CompanyA";
    var eventId = Guid.Parse("5c8ab5fa-f438-4ab4-8da4-9e5728c0ed43");
    DateTime transactionTime = CreateDataTimeFromString("2025-10-17 10:15:42.0000023");

    testMongoDbAccessManagerTemporalBulkPersister.RemoveUserToEntityMapping(null, user, entityType, entity, eventId, transactionTime);

    List<UserToEntityMappingDocument> allUserToEntityMappingDocuments = userToEntityMappingsCollection.Find(FilterDefinition<UserToEntityMappingDocument>.Empty)
        .SortBy(document => document.TransactionFrom)
        .ToList();
    Assert.AreEqual(5, allUserToEntityMappingDocuments.Count);
    Assert.AreEqual(CreateDataTimeFromString("2025-10-09 07:53:28.0000000"), allUserToEntityMappingDocuments[0].TransactionTo);
    Assert.AreEqual(DateTime.MaxValue, allUserToEntityMappingDocuments[1].TransactionTo);
    Assert.AreEqual(DateTime.MaxValue, allUserToEntityMappingDocuments[2].TransactionTo);
    Assert.AreEqual(DateTime.MaxValue, allUserToEntityMappingDocuments[3].TransactionTo);
    Assert.AreEqual(CreateDataTimeFromString("2025-10-17 10:15:42.0000022"), allUserToEntityMappingDocuments[4].TransactionTo);
    Assert.AreEqual(1, userStringifier.ToStringCallCount);
}
```

Plus EphemeralMongo is highly configurable, allowing different MongoDB versions, and single node vs replica set among many other options.

#### Creating Databases and Indexes Dynamically

One of the nice things about MongoDB as compared to relational databases, is that you don't need to explicitly setup schema, tables, indexes, etc... in advance of using the MongoDB database instance.  Databases, collections, and indexes are either created implicitly by the MongoDB client, or can be easily created explicitly at runtime.  Additionally the client methods to create indexes are idempotent, so you don't need to worry about whether an index already exists before attempting to create it.  In ApplicationAccess, on it's first use after construction the [MongoDbAccessManagerTemporalBulkPersister](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Persistence.MongoDb/MongoDbAccessManagerTemporalBulkPersister.cs) class...

1. Creates the database if it doesn't already exist (done implicitly in the constructor by instantiating an [IMongoDatabase](https://mongodb.github.io/mongo-csharp-driver/3.5.0/api/MongoDB.Driver/MongoDB.Driver.IMongoDatabase.html))
2. Creates any collections which don't already exist
3. Creates any required indexes on collections which don't already exist

...done via the following methods...

```C#
/// <summary>
/// Creates all AccessManager collections in the database if they don't already exist.
/// </summary>
protected void CreateCollections()
{
    List<String> allCollectionNames = new()
    {
        eventIdToTransactionTimeMapCollectionName,
        usersCollectionName,
        groupsCollectionName,
        userToGroupMappingsCollectionName,
        groupToGroupMappingsCollectionName,
        userToApplicationComponentAndAccessLevelMappingsCollectionName,
        groupToApplicationComponentAndAccessLevelMappingsCollectionName,
        entityTypesCollectionName,
        entitiesCollectionName,
        userToEntityMappingsCollectionName,
        groupToEntityMappingsCollectionName
    };
    foreach (String currentCollectionName in allCollectionNames)
    {
        CreateCollection(currentCollectionName);
    }
}

/// <summary>
/// Creates a collection in the database if it doesn't already exist.
/// </summary>
/// <param name="collectionName">The name of the collection to create.</param>
protected void CreateCollection(String collectionName)
{
    foreach (String currentCollectionName in database.ListCollectionNames().ToEnumerable())
    {
        if (currentCollectionName == collectionName)
        {
            return;
        }
    }
    try
    {
        database.CreateCollection(collectionName);
    }
    catch (Exception e)
    {
        throw new Exception($"Failed to create collection '{collectionName}' in MongoDB.", e);
    }
}

/// <summary>
/// Creates all indexes in the database if they don't already exist.
/// </summary>
protected void CreateIndexes()
{
    // 'EventIdToTransactionTimeMap' indexes
    var eventIdToTransactionTimeMapCollection = database.GetCollection<EventIdToTransactionTimeMappingDocument>(eventIdToTransactionTimeMapCollectionName);
    var eventIdToTransactionTimeMapCollectionEventIdIndexModel = new CreateIndexModel<EventIdToTransactionTimeMappingDocument>(Builders<EventIdToTransactionTimeMappingDocument>.IndexKeys.Ascending(document => document.EventId));
    eventIdToTransactionTimeMapCollection.Indexes.CreateOne(eventIdToTransactionTimeMapCollectionEventIdIndexModel);
    var eventIdToTransactionTimeMapCollectionTransactioIindexModel = new CreateIndexModel<EventIdToTransactionTimeMappingDocument>(Builders<EventIdToTransactionTimeMappingDocument>.IndexKeys
        .Descending(document => document.TransactionTime)
        .Descending(document => document.TransactionSequence));
    eventIdToTransactionTimeMapCollection.Indexes.CreateOne(eventIdToTransactionTimeMapCollectionTransactioIindexModel);
    // 'Users' indexes
    var usersCollection = database.GetCollection<UserDocument>(usersCollectionName);
    var usersCollectionUserIndexModel = new CreateIndexModel<UserDocument>(Builders<UserDocument>.IndexKeys
        .Ascending(document => document.User)
        .Ascending(document => document.TransactionTo));
    usersCollection.Indexes.CreateOne(usersCollectionUserIndexModel);
    CreateTransactionFieldIndex<UserDocument>(usersCollectionName);
    // 'Groups' indexes
    var groupsCollection = database.GetCollection<GroupDocument>(groupsCollectionName);
    var groupsCollectionUserIndexModel = new CreateIndexModel<GroupDocument>(Builders<GroupDocument>.IndexKeys
        .Ascending(document => document.Group)
        .Ascending(document => document.TransactionTo));
    groupsCollection.Indexes.CreateOne(groupsCollectionUserIndexModel);
    CreateTransactionFieldIndex<GroupDocument>(groupsCollectionName);
    // 'UserToGroupMappings' indexes
    var userToGroupMappingCollection = database.GetCollection<UserToGroupMappingDocument>(userToGroupMappingsCollectionName);
    var userToGroupMappingCollectionUserIndexModel = new CreateIndexModel<UserToGroupMappingDocument>(Builders<UserToGroupMappingDocument>.IndexKeys
        .Ascending(document => document.User)
        .Ascending(document => document.TransactionTo));
    userToGroupMappingCollection.Indexes.CreateOne(userToGroupMappingCollectionUserIndexModel);
    var userToGroupMappingCollectionGroupIndexModel = new CreateIndexModel<UserToGroupMappingDocument>(Builders<UserToGroupMappingDocument>.IndexKeys
        .Ascending(document => document.Group)
        .Ascending(document => document.TransactionTo));
    userToGroupMappingCollection.Indexes.CreateOne(userToGroupMappingCollectionGroupIndexModel);
    CreateTransactionFieldIndex<UserToGroupMappingDocument>(userToGroupMappingsCollectionName);
    // 'GroupToGroupMappings' indexes
    var groupToGroupMappingCollection = database.GetCollection<GroupToGroupMappingDocument>(groupToGroupMappingsCollectionName);
    var groupToGroupMappingCollectionFromGroupIndexModel = new CreateIndexModel<GroupToGroupMappingDocument>(Builders<GroupToGroupMappingDocument>.IndexKeys
        .Ascending(document => document.FromGroup)
        .Ascending(document => document.TransactionTo));
    groupToGroupMappingCollection.Indexes.CreateOne(groupToGroupMappingCollectionFromGroupIndexModel);
    var groupToGroupMappingCollectionToGroupIndexModel = new CreateIndexModel<GroupToGroupMappingDocument>(Builders<GroupToGroupMappingDocument>.IndexKeys
        .Ascending(document => document.ToGroup)
        .Ascending(document => document.TransactionTo));
    groupToGroupMappingCollection.Indexes.CreateOne(groupToGroupMappingCollectionToGroupIndexModel);
    CreateTransactionFieldIndex<GroupToGroupMappingDocument>(groupToGroupMappingsCollectionName);
    // 'UserToApplicationComponentAndAccessLevelMappings' indexes
    var userToApplicationComponentAndAccessLevelMappingCollection = database.GetCollection<UserToApplicationComponentAndAccessLevelMappingDocument>(userToApplicationComponentAndAccessLevelMappingsCollectionName); 
    var userToApplicationComponentAndAccessLevelMappingCollectionUserIndexModel = new CreateIndexModel<UserToApplicationComponentAndAccessLevelMappingDocument>(Builders<UserToApplicationComponentAndAccessLevelMappingDocument>.IndexKeys
        .Ascending(document => document.User)
        .Ascending(document => document.ApplicationComponent)
        .Ascending(document => document.AccessLevel)
        .Ascending(document => document.TransactionTo));
    userToApplicationComponentAndAccessLevelMappingCollection.Indexes.CreateOne(userToApplicationComponentAndAccessLevelMappingCollectionUserIndexModel);
    var userToApplicationComponentAndAccessLevelMappingCollectionApplicationComponentIndexModel = new CreateIndexModel<UserToApplicationComponentAndAccessLevelMappingDocument>(Builders<UserToApplicationComponentAndAccessLevelMappingDocument>.IndexKeys
        .Ascending(document => document.ApplicationComponent)
        .Ascending(document => document.AccessLevel)
        .Ascending(document => document.TransactionTo));
    userToApplicationComponentAndAccessLevelMappingCollection.Indexes.CreateOne(userToApplicationComponentAndAccessLevelMappingCollectionApplicationComponentIndexModel);
    CreateTransactionFieldIndex<UserToApplicationComponentAndAccessLevelMappingDocument>(userToApplicationComponentAndAccessLevelMappingsCollectionName);
    // 'GroupToApplicationComponentAndAccessLevelMappings' indexes
    var groupToApplicationComponentAndAccessLevelMappingCollection = database.GetCollection<GroupToApplicationComponentAndAccessLevelMappingDocument>(groupToApplicationComponentAndAccessLevelMappingsCollectionName);
    var groupToApplicationComponentAndAccessLevelMappingCollectionGroupIndexModel = new CreateIndexModel<GroupToApplicationComponentAndAccessLevelMappingDocument>(Builders<GroupToApplicationComponentAndAccessLevelMappingDocument>.IndexKeys
        .Ascending(document => document.Group)
        .Ascending(document => document.ApplicationComponent)
        .Ascending(document => document.AccessLevel)
        .Ascending(document => document.TransactionTo));
    groupToApplicationComponentAndAccessLevelMappingCollection.Indexes.CreateOne(groupToApplicationComponentAndAccessLevelMappingCollectionGroupIndexModel);
    var groupToApplicationComponentAndAccessLevelMappingCollectionApplicationComponentIndexModel = new CreateIndexModel<GroupToApplicationComponentAndAccessLevelMappingDocument>(Builders<GroupToApplicationComponentAndAccessLevelMappingDocument>.IndexKeys
        .Ascending(document => document.ApplicationComponent)
        .Ascending(document => document.AccessLevel)
        .Ascending(document => document.TransactionTo));
    groupToApplicationComponentAndAccessLevelMappingCollection.Indexes.CreateOne(groupToApplicationComponentAndAccessLevelMappingCollectionApplicationComponentIndexModel);
    CreateTransactionFieldIndex<GroupToApplicationComponentAndAccessLevelMappingDocument>(groupToApplicationComponentAndAccessLevelMappingsCollectionName);
    // 'EntityTypes' indexes
    var entityTypesCollection = database.GetCollection<EntityTypeDocument>(entityTypesCollectionName);
    var entityTypesCollectionEntityTypeIndexModel = new CreateIndexModel<EntityTypeDocument>(Builders<EntityTypeDocument>.IndexKeys
        .Ascending(document => document.EntityType)
        .Ascending(document => document.TransactionTo));
    entityTypesCollection.Indexes.CreateOne(entityTypesCollectionEntityTypeIndexModel);
    CreateTransactionFieldIndex<EntityTypeDocument>(entityTypesCollectionName);
    // 'Entities' indexes
    var entitiesCollection = database.GetCollection<EntityDocument>(entitiesCollectionName);
    var entitiesCollectionEntityIndexModel = new CreateIndexModel<EntityDocument>(Builders<EntityDocument>.IndexKeys
        .Ascending(document => document.EntityType)
        .Ascending(document => document.Entity)
        .Ascending(document => document.TransactionTo));
    entitiesCollection.Indexes.CreateOne(entitiesCollectionEntityIndexModel);
    CreateTransactionFieldIndex<EntityDocument>(entitiesCollectionName);
    // 'UserToEntityMappings' indexes
    var userToEntityMappingCollection = database.GetCollection<UserToEntityMappingDocument>(userToEntityMappingsCollectionName);
    var userToEntityMappingCollectionUserIndexModel = new CreateIndexModel<UserToEntityMappingDocument>(Builders<UserToEntityMappingDocument>.IndexKeys
        .Ascending(document => document.User)
        .Ascending(document => document.EntityType)
        .Ascending(document => document.Entity)
        .Ascending(document => document.TransactionTo));
    userToEntityMappingCollection.Indexes.CreateOne(userToEntityMappingCollectionUserIndexModel);
    var userToEntityMappingCollectionEntityTypeIndexModel = new CreateIndexModel<UserToEntityMappingDocument>(Builders<UserToEntityMappingDocument>.IndexKeys
        .Ascending(document => document.EntityType)
        .Ascending(document => document.Entity)
        .Ascending(document => document.TransactionTo));
    userToEntityMappingCollection.Indexes.CreateOne(userToEntityMappingCollectionEntityTypeIndexModel);
    CreateTransactionFieldIndex<UserToEntityMappingDocument>(userToEntityMappingsCollectionName);
    // 'GroupToEntityMappings' indexes
    var groupToEntityMappingCollection = database.GetCollection<GroupToEntityMappingDocument>(groupToEntityMappingsCollectionName);
    var groupToEntityMappingCollectionGroupIndexModel = new CreateIndexModel<GroupToEntityMappingDocument>(Builders<GroupToEntityMappingDocument>.IndexKeys
        .Ascending(document => document.Group)
        .Ascending(document => document.EntityType)
        .Ascending(document => document.Entity)
        .Ascending(document => document.TransactionTo));
    groupToEntityMappingCollection.Indexes.CreateOne(groupToEntityMappingCollectionGroupIndexModel);
    var groupToEntityMappingCollectionEntityTypeIndexModel = new CreateIndexModel<GroupToEntityMappingDocument>(Builders<GroupToEntityMappingDocument>.IndexKeys
        .Ascending(document => document.EntityType)
        .Ascending(document => document.Entity)
        .Ascending(document => document.TransactionTo));
    groupToEntityMappingCollection.Indexes.CreateOne(groupToEntityMappingCollectionEntityTypeIndexModel);
    CreateTransactionFieldIndex<GroupToEntityMappingDocument>(groupToEntityMappingsCollectionName);
}

/// <summary>
/// Creates a compound index on the TransactionFrom and TransactionTo fields of a collection holding documents derived from DocumentBase.
/// </summary>
/// <typeparam name="T">The type of documents (deriving from DocumentBase) held by the collection the index is being created on.</typeparam>
/// <param name="collectionName">The name of the collection to create the index on.</param>
protected void CreateTransactionFieldIndex<T>(String collectionName) where T : DocumentBase
{
    var collection = database.GetCollection<T>(collectionName);
    var collectionTransactionIndexModel = new CreateIndexModel<T>
    (
        Builders<T>.IndexKeys
            .Ascending(document => document.TransactionTo)
            .Ascending(document => document.TransactionFrom)
    );
    collection.Indexes.CreateOne(collectionTransactionIndexModel);
}
```

#### Transactions

Events in ApplicationAccess typically require 2 steps to persist into a database...

1. Either add a new row/document (in the case of an 'add' event), or update an existing row/document to end-date it (in the case of a 'remove' event).
2. Add a row/document to the EventIdToTransactionTimeMap table/collection.

In MongoDB, [operations on single documents are atomic](https://www.mongodb.com/docs/manual/core/transactions/#transactions), but operations across multiple are not without using [transactions](https://www.mongodb.com/docs/manual/core/transactions/).  Whilst using the [MongoDbAccessManagerTemporalBulkPersister](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Persistence.MongoDb/MongoDbAccessManagerTemporalBulkPersister.cs) without transactions is supported, it's not recommended as it could result in the database getting into an inconsistent state that can only be corrected by manual intervention (e.g. if a failure occurred between the 2 steps listed above and resulted in the first change being applied but not the second).

Executing changes within a transaction is straighforward... it can be implemented by creating a [IClientSessionHandle](https://mongodb.github.io/mongo-csharp-driver/2.22/html/T_MongoDB_Driver_IClientSessionHandle.htm) and calling the [WithTransaction()](https://mongodb.github.io/mongo-csharp-driver/2.22/html/M_MongoDB_Driver_IClientSession_WithTransaction__1.htm) method...

```C#
protected void RemoveUserToEntityMappingWithTransaction(TUser user, String entityType, String entity, Guid eventId, DateTime transactionTime)
{
    using (IClientSessionHandle session = mongoClient.StartSession())
    {
        session.WithTransaction<Object>((IClientSessionHandle s, CancellationToken ct) =>
        {
            RemoveUserToEntityMapping(session, user, entityType, entity, eventId, transactionTime);
            return new Object();
        });
    }
}
```

However, only replica-set instances of MongoDB support transactions... standalone instances do not.  But fortunately you can run a single node replica-set instance of MongoDB fairly simply using the [MongoDB Community docker image](https://hub.docker.com/r/mongodb/mongodb-community-server), which gives you a replica-set instance suitable for development testing...

1. Run the 'mongodb-community-server' docker image passing the 'replSet' parameter with the name of the replica set...

```
docker run --name mongodb -p 27017:27017 -d mongodb/mongodb-community-server:latest --replSet "myReplicaSet"
```

2. Log into the database with [mongosh](https://www.mongodb.com/docs/mongodb-shell/)...

```
mongosh "mongodb://{docker server host or IP}:27017"
```

3. Initialize the replica set...

```
rs.initiate()
rs.add("{docker server host or IP}:27017")
```

#### DateTime Precision

The [temporal data model](https://en.wikipedia.org/wiki/Temporal_database) in ApplicationAccess relies on keeping the full precision of fractional seconds of the [DateTime](https://learn.microsoft.com/en-us/dotnet/api/system.datetime.ticks?view=net-10.0) fields which store [event timestamps](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Persistence/Models/TemporalEventBufferItemBase.cs#L35).  By default .NET [DateTimes](https://learn.microsoft.com/en-us/dotnet/api/system.datetime.ticks?view=net-10.0) map to MongoDB [BSON Dates](https://www.mongodb.com/docs/manual/reference/bson-types/#date) which only offer millisecond precision, and hence weren't sufficient for ApplicationAccess.  However, using the [BsonDateTimeOptions](https://mongodb.github.io/mongo-csharp-driver/2.5/apidocs/html/T_MongoDB_Bson_Serialization_Attributes_BsonDateTimeOptionsAttribute.htm) attribute, you can have DateTimes stored as 64 bit integers which maintains the required precision.  I.e....

```C#
/// <summary>
/// Base for MongoDB documents.
/// </summary>
public abstract record DocumentBase
{
    public ObjectId _id;

    [BsonDateTimeOptions(Representation = BsonType.Int64, Kind = DateTimeKind.Utc)]
    public required DateTime TransactionFrom { get; init; }

    [BsonDateTimeOptions(Representation = BsonType.Int64, Kind = DateTimeKind.Utc)]
    public required DateTime TransactionTo { get; init; }
}
```

The only downside of this is that the DateTime fields are difficult to read when queried directly from MongoDB, e.g...

```
{
	"_id" : ObjectId("693e85b9681fb21ed2d1249f"),
	"TransactionFrom" : Long("639013019339711100"),
	"TransactionTo" : Long("3155378975999999999"),
	"User" : "38306d72-95be-49a8-8e7b-9e71141e4b25"
}
```

#### Snapshot Isolation

During performance and load testing of the MongoDB implementation, I tested the case where concurrent read and write operations overlap.  To do this I used a 3-node setup with an event cache node which only held a single event... this forces the read node to repeatedly reload the entire data state from MongoDB, which will eventually overlap with event writes from the wrier node.  The same test methodology was used to identify and resolve deadlocks with the SQL Server implementation.  However during the MongoDB testing, I encountered unusual errors where queries generated from the [MongoDbAccessManagerTemporalBulkPersister.Load() method](https://github.com/alastairwyse/ApplicationAccess/blob/15c32d8875683aa92a14044045df3e47d723b7cb/ApplicationAccess.Persistence.MongoDb/MongoDbAccessManagerTemporalBulkPersister.cs#L1134) would fail to return documents which had clearly been previously successfully written.  After quite a lot of searching I found [this Medium article](https://blog.meteor.com/mongodb-queries-dont-always-return-all-matching-documents-654b6594a827) which seemed to exactly describe the symptoms I was experiencing.

Seems that by default MongoDB can fail to return documents that fit within filter query criteria, if those documents are being updated at the time of querying.

The solution is to use [read concern 'snapshot'](https://www.mongodb.com/docs/manual/reference/read-concern-snapshot/) when querying, which (should!) guarantee to read the data as it appeared at the start of querying, over the duration that the query proceeds.  This can be set as part of the [ClientSessionOptions](https://mongodb.github.io/mongo-csharp-driver/2.22/html/T_MongoDB_Driver_ClientSessionOptions.htm) class...

```C#
IClientSessionHandle session = null;
if (useTransactions == true)
{
    var clientSessionOptions = new ClientSessionOptions();
    clientSessionOptions.CausalConsistency = false;
    clientSessionOptions.Snapshot = true;
    session = mongoClient.StartSession(clientSessionOptions);
}
try
{
    accessManagerToLoadTo.Clear();
    foreach (String currentUser in GetUsers(session, stateTime))
    {
        accessManagerToLoadTo.AddUser(userStringifier.FromString(currentUser));
    }
    // etc...
```

#### Understanding Query Performance

Most of the below was gleaned from trial and error rather than reading documentation.  Firstly, you can get the query statistics from MongoDB by adding this to the query...

```JSON
.explain("executionStats")
```

When interpreting query plans in relational databases, you usually pay significant attention to the method by which a table is accessed (e.g. table scan, index scan or index seek).  In MongoDB this seems less important... i.e. in a fully tuned/efficient query you'll often see 'IXSCAN' (index scan) denoted as the 'stage' of access, e.g...

```JSON
"stage" : "IXSCAN"
```

... but similarly in an inefficient query which scans all documents in the default '_id' index on a collection (which is what seems to happen when MongoDB decides to scan all documents in a collection), you see the same 'IXSCAN' stage in the statistics.  What's more indicative of performance are the 'totalKeysExamined', and 'totalDocsExamined' values in the 'executionStats' section of the statistics... ideally you want these to match the numbers of documents returned by the query, e.g. in the below example which returns a single document from a collection containing 8215 documents...

```JSON
"totalKeysExamined" : 1,
"totalDocsExamined" : 1,
```

Conversely, you don't want these numbers to be large... extending to the worst case where they match the number of documents in the collection (meaning the entire collection is being scanned to service the query)... e.g. the below statistics for the same collection...

```JSON
"totalKeysExamined" : 8215,
"totalDocsExamined" : 8215,
```
