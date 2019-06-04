<!--
GENERATED FILE - DO NOT EDIT
This file was generated by [MarkdownSnippets](https://github.com/SimonCropp/MarkdownSnippets).
Source File: /doco/mdsource/configuration.source.md
To change this file edit the source file and then run MarkdownSnippets.
-->
# Configuration

Enabling is done via registering in a container.

Configuration requires an instance of `Microsoft.EntityFrameworkCore.Metadata.IModel`, hence a DbContext instance is required at configuration time. As such a DbContext needs to be instantiated and disposed for the purposes of IModel construction. EF does not allow access to the model unless 

The container registration can be via addin to a [IServiceCollection](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.iservicecollection):

<!-- snippet: RegisterInContainerServiceCollection -->
```cs
public static void RegisterInContainer<TDbContext>(
    IServiceCollection services,
    TDbContext dbContext,
    DbContextFromUserContext<TDbContext> dbContextFromUserContext,
    GlobalFilters filters = null)
    where TDbContext : DbContext
```
<sup>[snippet source](/src/GraphQL.EntityFramework/EfGraphQLConventions.cs#L34-L41)</sup>
<!-- endsnippet -->

Usage:

<!-- snippet: RegisterInContainerServiceCollectionUsage -->
```cs
var builder = new DbContextOptionsBuilder();
builder.UseSqlServer("fake");
using (var context = new MyDbContext(builder.Options))
{
    EfGraphQLConventions.RegisterInContainer(
        serviceCollection,
        dbContext: context,
        dbContextFromUserContext: userContext => (MyDbContext) userContext);
}
```
<sup>[snippet source](/src/Snippets/Configuration.cs#L9-L21)</sup>
<!-- endsnippet -->

Or via an Action.

<!-- snippet: RegisterInContainerAction -->
```cs
public static void RegisterInContainer<TDbContext>(
    Action<Type, object> register,
    TDbContext dbContext,
    DbContextFromUserContext<TDbContext> dbContextFromUserContext,
    GlobalFilters filters = null)
    where TDbContext : DbContext
```
<sup>[snippet source](/src/GraphQL.EntityFramework/EfGraphQLConventions.cs#L10-L17)</sup>
<!-- endsnippet -->

Then the usage entry point `IEfGraphQLService` can be resolved via [dependency injection in GraphQL.net](https://graphql-dotnet.github.io/docs/guides/advanced#dependency-injection) to be used in `ObjectGraphType`s when adding query fields.


## DocumentExecuter

The default GraphQL `DocumentExecuter` uses [Task.WhenAll](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.task.whenall) to resolve async fields. This can result in multiple EF queries being executed on different threads and being resolved out of order. In this scenario the following exception will be thrown.

> Message: System.InvalidOperationException : A second operation started on this context before a previous operation completed. This is usually caused by different threads using the same instance of DbContext, however instance members are not guaranteed to be thread safe. This could also be caused by a nested query being evaluated on the client, if this is the case rewrite the query avoiding nested invocations.

To avoid this a custom implementation of `DocumentExecuter` but be used that uses `SerialExecutionStrategy` when the operation type is `OperationType.Query`. There is one included in this library named `EfDocumentExecuter`:

<!-- snippet: EfDocumentExecuter.cs -->
```cs
using GraphQL.Execution;
using GraphQL.Language.AST;

namespace GraphQL.EntityFramework
{
    public class EfDocumentExecuter :
        DocumentExecuter
    {
        protected override IExecutionStrategy SelectExecutionStrategy(ExecutionContext context)
        {
            Guard.AgainstNull(nameof(context), context);
            if (context.Operation.OperationType == OperationType.Query)
            {
                return new SerialExecutionStrategy();
            }
            return base.SelectExecutionStrategy(context);
        }
    }
}
```
<sup>[snippet source](/src/GraphQL.EntityFramework/EfDocumentExecuter.cs#L1-L19)</sup>
<!-- endsnippet -->


## Connection Types

GraphQL enables paging via [Connections](https://graphql.org/learn/pagination/#complete-connection-model). When using Connections in GraphQL.net it is [necessary to register several types in the container](https://github.com/graphql-dotnet/graphql-dotnet/issues/451#issuecomment-335894433):

```csharp
services.AddTransient(typeof(ConnectionType<>));
services.AddTransient(typeof(EdgeType<>));
services.AddSingleton<PageInfoType>();
```

There is a helper methods to perform the above:

```csharp
EfGraphQLConventions.RegisterConnectionTypesInContainer(IServiceCollection services);
```

or

```csharp
EfGraphQLConventions.RegisterConnectionTypesInContainer(Action<Type> register)
```


## DependencyInjection and ASP.Net Core

As with GraphQL .net, GraphQL.EntityFramework makes no assumptions on the container or web framework it is hosted in. However given [Microsoft.Extensions.DependencyInjection](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection) and [ASP.Net Core](https://docs.microsoft.com/en-us/aspnet/core/) are the most likely usage scenarios, the below will address those scenarios explicitly.

See the GraphQL .net [documentation for ASP.Net Core](https://graphql-dotnet.github.io/docs/getting-started/dependency-injection#aspnet-core) and the [ASP.Net Core sample](https://github.com/graphql-dotnet/examples/tree/master/src/AspNetCoreCustom/Example).

The Entity Framework Data Context instance is generally [scoped per request](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection#service-lifetimes-and-registration-options). This can be done in the [Startup.ConfigureServices method](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/startup#the-configureservices-method):

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddScoped(
          provider => MyDbContextBuilder.BuildDbContext());
    }
}
```

Entity Framework also provides [several helper methods](https://docs.microsoft.com/en-us/ef/core/miscellaneous/configuring-dbcontext#using-dbcontext-with-dependency-injection) to control a DbContexts lifecycle. For example:

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddDbContext<MyDbContext>(
          provider => DbContextBuilder.BuildDbContext());
    }
}
```

See also [EntityFrameworkServiceCollectionExtensions](https://docs.microsoft.com/en-us/ef/core/api/microsoft.extensions.dependencyinjection.entityframeworkservicecollectionextensions)

With the DbContext existing in the container, it can be resolved in the controller that handles the GraphQL query:

<!-- snippet: GraphQlController -->
```cs
[Route("[controller]")]
[ApiController]
public class GraphQlController :
    Controller
{
    IDocumentExecuter executer;
    ISchema schema;

    public GraphQlController(ISchema schema, IDocumentExecuter executer)
    {
        this.schema = schema;
        this.executer = executer;
    }

    [HttpPost]
    public Task<ExecutionResult> Post(
        [BindRequired, FromBody] PostBody body,
        [FromServices] GraphQlEfSampleDbContext dbContext,
        CancellationToken cancellation)
    {
        return Execute(dbContext, body.Query, body.OperationName, body.Variables, cancellation);
    }

    public class PostBody
    {
        public string OperationName;
        public string Query;
        public JObject Variables;
    }

    [HttpGet]
    public Task<ExecutionResult> Get(
        [FromQuery] string query,
        [FromQuery] string variables,
        [FromQuery] string operationName,
        [FromServices] GraphQlEfSampleDbContext dbContext,
        CancellationToken cancellation)
    {
        var jObject = ParseVariables(variables);
        return Execute(dbContext, query, operationName, jObject, cancellation);
    }

    async Task<ExecutionResult> Execute(
        GraphQlEfSampleDbContext dbContext,
        string query,
        string operationName,
        JObject variables,
        CancellationToken cancellation)
    {
        var options = new ExecutionOptions
        {
            Schema = schema,
            Query = query,
            OperationName = operationName,
            Inputs = variables?.ToInputs(),
            UserContext = dbContext,
            CancellationToken = cancellation,
#if (DEBUG)
            ExposeExceptions = true,
            EnableMetrics = true,
#endif
        };

        var result = await executer.ExecuteAsync(options);

        if (result.Errors?.Count > 0)
        {
            Response.StatusCode = (int) HttpStatusCode.BadRequest;
        }

        return result;
    }

    static JObject ParseVariables(string variables)
    {
        if (variables == null)
        {
            return null;
        }

        try
        {
            return JObject.Parse(variables);
        }
        catch (Exception exception)
        {
            throw new Exception("Could not parse variables.", exception);
        }
    }
}
```
<sup>[snippet source](/src/SampleWeb/GraphQlController.cs#L11-L102)</sup>
<!-- endsnippet -->

Note that the instance of the DbContext is passed to the [GraphQL .net User Context](https://graphql-dotnet.github.io/docs/getting-started/user-context).

The same instance of the DbContext can then be accessed in the `resolve` delegate by casting the `ResolveFieldContext.UserContext` to the DbContext type:

<!-- snippet: QueryUsedInController -->
```cs
public class Query :
    QueryGraphType<GraphQlEfSampleDbContext>
{
    public Query(IEfGraphQLService<GraphQlEfSampleDbContext> efGraphQlService) :
        base(efGraphQlService)
    {
        AddQueryField(
            name: "companies",
            resolve: context => context.DbContext.Companies);
```
<sup>[snippet source](/src/SampleWeb/Query.cs#L6-L18)</sup>
<!-- endsnippet -->


## Multiple DbContexts

Multiple different DbContext types can be registered and used.


### UserContext

A user context that exposes both types.

<!-- snippet: MultiUserContext -->
```cs
public class UserContext
{
    public DbContext1 DbContext1;
    public DbContext2 DbContext2;
}
```
<sup>[snippet source](/src/Tests/MultiContextTests/MultiContextTests.cs#L109-L115)</sup>
<!-- endsnippet -->


### Register in container

Register both DbContext types in the container and include how those instance can be extracted from the GraphQL context:

<!-- snippet: RegisterMultipleInContainer -->
```cs
EfGraphQLConventions.RegisterInContainer(
    services,
    dbContext1,
    userContext => ((UserContext) userContext).DbContext1);
EfGraphQLConventions.RegisterInContainer(
    services,
    dbContext2,
    userContext => ((UserContext) userContext).DbContext2);
```
<sup>[snippet source](/src/Tests/MultiContextTests/MultiContextTests.cs#L70-L79)</sup>
<!-- endsnippet -->


### ExecutionOptions

Use the user type to pass in both DbContext instances.


<!-- snippet: MultiExecutionOptions -->
```cs
var executionOptions = new ExecutionOptions
{
    Schema = schema,
    Query = query,
    UserContext = new UserContext
    {
        DbContext1 = dbContext1,
        DbContext2 = dbContext2
    }
};
```
<sup>[snippet source](/src/Tests/MultiContextTests/MultiContextTests.cs#L85-L96)</sup>
<!-- endsnippet -->


### Query

Use both DbContexts in a Query:

<!-- snippet: MultiContextQuery.cs -->
```cs
using GraphQL.EntityFramework;
using GraphQL.Types;

public class MultiContextQuery :
    ObjectGraphType
{
    public MultiContextQuery(
        IEfGraphQLService<DbContext1> efGraphQlService1,
        IEfGraphQLService<DbContext2> efGraphQlService2)
    {
        efGraphQlService1.AddSingleField(
            graph: this,
            name: "entity1",
            resolve: context =>
            {
                var userContext = (UserContext) context.UserContext;
                return userContext.DbContext1.Entities;
            });
        efGraphQlService2.AddSingleField(
            graph: this,
            name: "entity2",
            resolve: context =>
            {
                var userContext = (UserContext) context.UserContext;
                return userContext.DbContext2.Entities;
            });
    }
}
```
<sup>[snippet source](/src/Tests/MultiContextTests/MultiContextQuery.cs#L1-L28)</sup>
<!-- endsnippet -->


### GraphType

Use a DbContext in a Graph:

<!-- snippet: Entity1Graph.cs -->
```cs
using GraphQL.EntityFramework;

public class Entity1Graph :
    EfObjectGraphType<DbContext1, Entity1>
{
    public Entity1Graph(IEfGraphQLService<DbContext1> graphQlService) :
        base(graphQlService)
    {
        Field(x => x.Id);
        Field(x => x.Property);
    }
}
```
<sup>[snippet source](/src/Tests/MultiContextTests/Graphs/Entity1Graph.cs#L1-L12)</sup>
<!-- endsnippet -->


## Testing the GraphQlController

The `GraphQlController` can be tested using the [ASP.NET Integration tests](https://docs.microsoft.com/en-us/aspnet/core/test/integration-tests) via the [Microsoft.AspNetCore.Mvc.Testing NuGet package](https://www.nuget.org/packages/Microsoft.AspNetCore.Mvc.Testing).

<!-- snippet: GraphQlControllerTests -->
```cs
public class GraphQlControllerTests :
    XunitLoggingBase
{
    static HttpClient client;

    static WebSocketClient websocketClient;

    static GraphQlControllerTests()
    {
        var server = GetTestServer();
        client = server.CreateClient();
        websocketClient = server.CreateWebSocketClient();
        websocketClient.ConfigureRequest =
            request => { request.Headers["Sec-WebSocket-Protocol"] = "graphql-ws"; };
    }

    [Fact]
    public async Task Get()
    {
        var query = @"
{
  companies
  {
    id
  }
}";
        var response = await ClientQueryExecutor.ExecuteGet(client, query);
        response.EnsureSuccessStatusCode();
        var result = await response.Content.ReadAsStringAsync();
        Assert.Contains(
            "{\"companies\":[{\"id\":1},{\"id\":4},{\"id\":6},{\"id\":7}]}",
            result);
    }

    [Fact]
    public async Task Get_single()
    {
        var query = @"
query ($id: ID!)
{
  company(id:$id)
  {
    id
  }
}";
        var variables = new
        {
            id = "1"
        };

        var response = await ClientQueryExecutor.ExecuteGet(client, query, variables);
        response.EnsureSuccessStatusCode();
        var result = await response.Content.ReadAsStringAsync();
        Assert.Contains(@"{""data"":{""company"":{""id"":1}}}", result);
    }

    [Fact]
    public async Task Get_variable()
    {
        var query = @"
query ($id: ID!)
{
  companies(ids:[$id])
  {
    id
  }
}";
        var variables = new
        {
            id = "1"
        };

        var response = await ClientQueryExecutor.ExecuteGet(client, query, variables);
        response.EnsureSuccessStatusCode();
        var result = await response.Content.ReadAsStringAsync();
        Assert.Contains("{\"companies\":[{\"id\":1}]}", result);
    }

    [Fact]
    public async Task Get_companies_paging()
    {
        var after = 1;
        var query = @"
query {
  companiesConnection(first:2, after:""" + after + @""") {
    edges {
      cursor
      node {
        id
      }
    }
    pageInfo {
      endCursor
      hasNextPage
    }
  }
}";
        var response = await ClientQueryExecutor.ExecuteGet(client, query);
        response.EnsureSuccessStatusCode();
        var result = JObject.Parse(await response.Content.ReadAsStringAsync());

        var page = result.SelectToken("..data..companiesConnection..edges[0].cursor")
            .Value<string>();
        Assert.NotEqual(after.ToString(), page);
    }

    [Fact]
    public async Task Get_employee_summary()
    {
        var query = @"
query {
  employeeSummary {
    companyId
    averageAge
  }
}";
        var response = await ClientQueryExecutor.ExecuteGet(client, query);
        response.EnsureSuccessStatusCode();
        var result = JObject.Parse(await response.Content.ReadAsStringAsync());

        var expected = JObject.FromObject(new
        {
            data = new
            {
                employeeSummary = new[]
                {
                    new {companyId = 1, averageAge = 28.0},
                    new {companyId = 4, averageAge = 34.0}
                }
            }
        });
        Assert.Equal(expected.ToString(), result.ToString());
    }

    [Fact]
    public async Task Post()
    {
        var query = @"
{
  companies
  {
    id
  }
}";
        var response = await ClientQueryExecutor.ExecutePost(client, query);
        var result = await response.Content.ReadAsStringAsync();
        Assert.Contains(
            "{\"companies\":[{\"id\":1},{\"id\":4},{\"id\":6},{\"id\":7}]}",
            result);
        response.EnsureSuccessStatusCode();
    }

    [Fact]
    public async Task Post_variable()
    {
        var query = @"
query ($id: ID!)
{
  companies(ids:[$id])
  {
    id
  }
}";
        var variables = new
        {
            id = "1"
        };
        var response = await ClientQueryExecutor.ExecutePost(client, query, variables);
        var result = await response.Content.ReadAsStringAsync();
        Assert.Contains("{\"companies\":[{\"id\":1}]}", result);
        response.EnsureSuccessStatusCode();
    }

    [Fact]
    public Task Should_subscribe_to_companies()
    {
        var resetEvent = new AutoResetEvent(false);

        var result = new GraphQLHttpSubscriptionResult(
            new Uri("http://example.com/graphql"),
            new GraphQLRequest
            {
                Query = @"
subscription 
{
  companyChanged 
  {
    id
  }
}"
            },
            websocketClient);

        result.OnReceive +=
            res =>
            {
                if (res == null)
                {
                    return;
                }
                Assert.Null(res.Errors);

                if (res.Data != null)
                {
                    resetEvent.Set();
                }
            };

        var cancellationSource = new CancellationTokenSource();

        var task = result.StartAsync(cancellationSource.Token);

        Assert.True(resetEvent.WaitOne(TimeSpan.FromSeconds(10)));

        cancellationSource.Cancel();

        return task;
    }

    static TestServer GetTestServer()
    {
        var hostBuilder = new WebHostBuilder();
        hostBuilder.UseStartup<Startup>();
        return new TestServer(hostBuilder);
    }

    public GraphQlControllerTests(ITestOutputHelper output) : 
        base(output)
    {
    }
}
```
<sup>[snippet source](/src/SampleWeb.Tests/GraphQlControllerTests.cs#L13-L247)</sup>
<!-- endsnippet -->


## GraphQlExtensions

The `GraphQlExtensions` class exposes some helper methods:


### ExecuteWithErrorCheck

Wraps the `DocumentExecuter.ExecuteAsync` to throw if there are any errors.

<!-- snippet: ExecuteWithErrorCheck -->
```cs
public static async Task<ExecutionResult> ExecuteWithErrorCheck(this IDocumentExecuter documentExecuter, ExecutionOptions executionOptions)
{
    Guard.AgainstNull(nameof(documentExecuter), documentExecuter);
    Guard.AgainstNull(nameof(executionOptions), executionOptions);
    var executionResult = await documentExecuter.ExecuteAsync(executionOptions);

    var errors = executionResult.Errors;
    if (errors != null && errors.Count > 0)
    {
        if (errors.Count == 1)
        {
            throw errors.First();
        }

        throw new AggregateException(errors);
    }

    return executionResult;
}
```
<sup>[snippet source](/src/GraphQL.EntityFramework/GraphQlExtensions.cs#L9-L31)</sup>
<!-- endsnippet -->