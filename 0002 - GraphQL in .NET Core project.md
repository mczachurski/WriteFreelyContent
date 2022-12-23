# GraphQL in .NET Core project

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0003.png)

> This is the article created at Oct 10, 2017 and moved from Medium.

In last few days I worked on integration GraphQL into my .NET Core project. I would like to share with you with all steps that I had to do.
<!--more-->

---

I found only one library which we can use when we want to create GraphQL API in .NET Core project: [http://graphql-dotnet.github.io/graphql-dotnet/](http://graphql-dotnet.github.io/graphql-dotnet/).

Unfortunately documentation of that project isn’t extended. However there is separate github project where we can find more information: [https://github.com/JacekKosciesza/StarWars](https://github.com/JacekKosciesza/StarWars).

Based on that information I successfully created GraphQL endpoint which serves the same information which produces my REST API but now my clients can execute more queries in one request. And this is real performance boost.

### Add nuget package

First we have to add nuget package to our `csproj` file. We can execute in command line action: `dotnet add package GraphQL`. After that command in project file we should have a new entry:

```xml
<PackageReference Include="GraphQL" Version="0.17.3" />
```

### Controller

We can start our work from controller. GraphQL requires only one HTTP endpoint with POST method. Below there is simple example of controller.

```cs
[Route("api/graphql")]
public class GraphQLController : Controller
{
    private readonly GraphQLQuery _graphQLQuery;
    private readonly IDocumentExecuter _documentExecuter;
    private readonly ISchema _schema;

    public GraphQLController(GraphQLQuery graphQLQuery, IDocumentExecuter documentExecuter, ISchema schema)
    {
        _graphQLQuery = graphQLQuery;
        _documentExecuter = documentExecuter;
        _schema = schema;
    }

    [HttpPost]
    public async Task<IActionResult> Post([FromBody] GraphQLParameter query)
    {
        var executionOptions = new ExecutionOptions { Schema = _schema, Query = query.Query, UserContext = User };
        var result = await _documentExecuter.ExecuteAsync(executionOptions).ConfigureAwait(false);

        if (result.Errors?.Count > 0)
        {
            return BadRequest(result.Errors);
        }

        return Ok(result);
    }
}
```

In our controller we have four things which are mandatory for .NET GraphQL:

- `GraphQLQuery` — all query roots which we can execute. This class we have to create.
- `IDocumentExecuter` — interface from .NET GrapQL. It is responsible for executing queries.
- `ISchema` — interfece which we have to implement to tell GraphQL about all our types which can be returned.
- `GraphQLParameter` — class which stores query parameters.

#### GraphQLParameter

The simples one is last class. It stores one property for GraphQL query.

```cs
public class GraphQLParameter
{
    public string Query { get; set; }
}
```

#### IDocumentExecuter

This is interface from .NET GraphQL. Also there is a default implementation of that interface. We have to do only one thing with it — add to our dependency injection container. For autofac we have to add line:

```cs
builder.RegisterType<DocumentExecuter>().As<IDocumentExecuter>();
```

#### ISchema

If we want to inject services/repositories to our GraphQL types we have to implement that interface. It’s very simple.

```cs
public class GraphQLSchema : Schema
{
    public GraphQLSchema(Func<Type, GraphType> resolveType)
        : base(resolveType)
    {
        Query = (GraphQLQuery)resolveType(typeof(GraphQLQuery));
    }
}
```

Also we have to add to dependency injection `ISchema` implementation (and resolveType constructor parameter).

```cs
builder.RegisterType<GraphQLSchema>().As<ISchema>();
builder.Register<Func<Type, GraphType>>(c => 
{
    var context = c.Resolve<IComponentContext>();
    return t => {
        var res = context.Resolve(t);
        return (GraphType)res;
    };
});
```

#### GraphQLQuery

Last thing is class which will be responsible for storing all of our query roots. I decided to move implementation of the queries to separate resolvers. Thanks to this I don’t have one big class which is responsible for different things. My `GraphQLQuery` looks like on below snippet.

```cs
public class GraphQLQuery : ObjectGraphType
{
    public GraphQLQuery(IServiceProvider serviceProvider)
    {
        var type = typeof(IResolver);
        var resolversTypes = AppDomain.CurrentDomain.GetAssemblies()
            .SelectMany(s => s.GetTypes())
            .Where(p => type.IsAssignableFrom(p));

        foreach(var resolverType in resolversTypes)
        {
            var resolverTypeInterface = resolverType.GetInterfaces().Where(x => x != type).FirstOrDefault();
            if(resolverTypeInterface != null)
            {
                var resolver = serviceProvider.GetService(resolverTypeInterface) as IResolver;
                resolver.Resolve(this);
            }
        }
    }
}
```

So I’m using reflection to get all classes which implement `IResolver` interface. Then I’m creating instances thanks to `ServiceProvider` which exists in .NET Core. Resolvers are described below.

### Resolvers

Resolvers are classes which are responsible for retrieve data which should be returned by specific query. All my resolvers are marked by `IResolver` interface, which has only one method.

```cs
public interface IResolver
{
    void Resolve(GraphQLQuery graphQLQuery);
}
```

Below there is an implementation of one of the resolver.

```cs
public class GroupsResolver : Resolver, IGroupsResolver
{
    private readonly IGroupsService _groupsService;

    public GroupsResolver(IGroupsService groupsService)
    {
        _groupsService = groupsService;
    }

    public void Resolve(GraphQLQuery graphQLQuery)
    {
        graphQLQuery.Field<ListGraphType<GroupType>>(
            "groups",
            resolve: context => _groupsService.GetGroupsAsync()
        );
    }
}
```

Above resolver is responsible only for one thing. It registers one query: `groups`. And that query returns list of groups.

We can see that resolver uses `IGroupsServices`. That service can retrieve groups from database for example.

Here we have also `GroupType` class which is really important from GraphQL perspective. It stores information which GraphQL uses to introspection (whole information about API schema is built thanks to this).

All resolvers can be registered in dependency injection container like that:

```cs
var serviceAssembly = typeof(IResolver).GetTypeInfo().Assembly;

builder.RegisterAssemblyTypes(serviceAssembly)
    .Where(t => t.Name.EndsWith("Resolver"))
    .AsImplementedInterfaces()
    .InstancePerLifetimeScope();
```

### Add GraphQL types

The last thing are GraphQL types. Type system is the heart of GraphQL. Below there is a simple example of GraphQL type.

```cs
public class GroupType : ObjectGraphType<GroupDto>, IGraphQLType
{
    public GroupType()
    {
        Field(x => x.Name).Description("The group name.");
        Field(x => x.SvgIcon).Description("The icon of group.");
    }
}
```

Above type uses `GroupDto`. That class is returned by `IGroupService` in resolver (described above) and transformed to `GroupType` class by .NET GraphQL.

```cs
public class GroupDto
{
    public string Name { get; set; }
    public string SvgIcon { get; set; }
}
```

All types should be registered in dependency injection (especially when we inject to the type services which will resolve property data). Below there is a code which I used to register my GraphQL types.

```cs
var serviceAssembly = typeof(IGraphQLType).GetTypeInfo().Assembly;
builder.RegisterAssemblyTypes(serviceAssembly)
    .Where(t => t.GetInterfaces()
        .Any(i => i.IsAssignableFrom(typeof (IGraphQLType))))
    .AsSelf();
```

All my types are marked by `IGraphType` interface (which is empty). Thanks to this I can register easily all of types in DI container.

---

You can find more source code connected to GraphQL in one of my [projec](https://github.com/BibliothecaTeam/Bibliotheca.Server.Gateway/tree/master/src/Bibliotheca.Server.Gateway.Api)t on github.