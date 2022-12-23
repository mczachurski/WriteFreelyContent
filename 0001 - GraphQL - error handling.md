# GraphQL - error handling

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0001.png)

> This is the article created at Oct 10, 2017 and moved from Medium.

I’m big fan of GraphQL. It seems that it will be next industry standard for API. It’s really easy to implement (at least in .net world) and it delivers many great features. Main issue which GraphQL solves is minimization of requests to the API server from web/native applications. Thus it’s great for SPA or native applications. More details about GraphQL you can find on the [official page](http://graphql.org).

<!--more-->

One of the issues which I had with GraphQL concept was errors/authorization handling. In the REST API each requests is run separately so it’s really easy to return information to the client about request status (we can use status codes e.g. 403 Forbidden, 404 NotFound etc.). We cannot use the same behaviour in GraphQL, becouse if one of the query fails we shouldn’t change status code for whole request. Some of queries in one request can be resolved properly some may fails, but still we have to return 200 OK status code. That’s why we have to figure out something else.

First step which I done when I come across that problem was preparing JSON which GraphQL should return.

```json
{
  "data": {
    "project": {
      "statusCode": "Success",
      "errorMessage": null,
      "data": {
        "id": "projectA",
        "name": "Project A",
        "visibleBranches": [
          "Latest"
        ],
        "tags": [
          "Data admin"
        ]
      }
    },
    "projects": {
      "statusCode": "Success",
      "errorMessage": null,
      "data": {
        "allResults": 3,
        "results": [
          {
            "name": "Project A",
            "id": "project-a"
          },
          {
            "name": "Project B",
            "id": "project-b"
          },
          {
            "name": "Project C",
            "id": "project-c"
          }
        ]
      }
    }
  }
}
```

Above we have JSON which was generated based on two queries (`project` and `projects`). Both are executed successfully thus we have in both queries `StatusCode` property with `Success` value. When one of query fails we should get response like in schema below.

```json
{
  "data": {
    "project": {
      "statusCode": "NotFound",
      "errorMessage": "Resource not found",
      "data": null
    },
    "projects": {
      "statusCode": "Success",
      "errorMessage": null,
      "data": {
        "allResults": 3,
        "results": [
          {
            "name": "Project A",
            "id": "project-a"
          },
          {
            "name": "Project B",
            "id": "project-b"
          },
          {
            "name": "Project C",
            "id": "project-c"
          }
        ]
      }
    }
  }
}
```

Generally I boxed response which I want to return by resolver by class which contains properties:

*   `StatusCode` — simple code with status code — similar like HTTP status codes but written as a string like: `NotFound`, `Forbidden` etc.
*   `ErrorMessage` — more detailed information about error if status code is not `Success`.
*   `Data` — main data which should be returned by query.

I created `ResponseGraphType` class to represent type field in GraphQL.

```cs
public class ResponseGraphType<TGraphType> : ObjectGraphType<Response> where TGraphType : GraphType
{
    public ResponseGraphType()
    {
        Name = $"Response{typeof(TGraphType).Name}";

        Field(x => x.StatusCode, nullable: true).Description("Status code of the request.");
        Field(x => x.ErrorMessage, nullable: true).Description("Error message if requests fails.");

        Field<TGraphType>(
            "data",
            "Data returned by query.",
            resolve: context => context.Source.Data
        );
    }
}
```

Also we have to create `Response` class which should be returned by our resolvers.

```cs
public class Response
{
    public object Data { get; set; }
    public string StatusCode { get; set; }
    public string ErrorMessage { get; set; }

    public Response(object data)
    {
        StatusCode = "Success";
        Data = data;
    }

    public Response(string statusCode, string errorMessage)
    {
        StatusCode = statusCode;
        ErrorMessage = errorMessage;
    }
}
```

Now we can use both above classes in resolver. Our GraphQL field will be `ResponseGraphType<>` type, and resolver should return `Response` object.

```cs
public class BranchesResolver : IBranchesResolver
{
    private readonly IBranchesService _branchesService;

    public BranchesResolver(IBranchesService branchesService)
    {
        _branchesService = branchesService;
    }

    public void Resolve(GraphQLQuery graphQLQuery)
    {
        graphQLQuery.Field<ResponseGraphType<BranchType>>(
            "branch",
            arguments: new QueryArguments(
                new QueryArgument<NonNullGraphType<StringGraphType>> { Name = "projectId", Description = "id of the project" },
                new QueryArgument<NonNullGraphType<StringGraphType>> { Name = "branchName", Description = "name of the branch" }
            ),
            resolve: context => { 
                var projectId = context.GetArgument<string>("projectId");
                var branchName = context.GetArgument<string>("branchName");
                var branch = _branchesService.GetBranchAsync(projectId, branchName).GetAwaiter().GetResult();

                if(branch == null) 
                {
                    return new Response("NotFound", "Resource not found");
                }

                return new Response(branch);
            }
        );
    }
}
```

Also we can create base class for our resolvers (similar class to `Controller` class in MVC).

```cs
public class Resolver
{
    public Response Response(object data)
    {
        return new Response(data);
    }

    public Response Error(GraphQLError error)
    {
        return new Response(error.StatusCode, error.ErrorMessage);
    }

    public Response AccessDeniedError()
    {
        var error = new AccessDeniedError();
        return new Response(error.StatusCode, error.ErrorMessage);
    }

    public Response NotFoundError(string id)
    {
        var error = new NotFoundError(id);
        return new Response(error.StatusCode, error.ErrorMessage);
    }
}
```

After this we can inherite class where we have our queries from this new class and modify a little bit our resolver. Now it’s even more readable.

```cs
public class BranchesResolver : Resolver, IBranchesResolver
{
    private readonly IBranchesService _branchesService;

    public BranchesResolver(IBranchesService branchesService)
    {
        _branchesService = branchesService;
    }

    public void Resolve(GraphQLQuery graphQLQuery)
    {
        graphQLQuery.Field<ResponseGraphType<BranchType>>(
            "branch",
            arguments: new QueryArguments(
                new QueryArgument<NonNullGraphType<StringGraphType>> { Name = "projectId", Description = "id of the project" },
                new QueryArgument<NonNullGraphType<StringGraphType>> { Name = "branchName", Description = "name of the branch" }
            ),
            resolve: context => { 
                var projectId = context.GetArgument<string>("projectId");
                var branchName = context.GetArgument<string>("branchName");
                var branch = _branchesService.GetBranchAsync(projectId, branchName).GetAwaiter().GetResult();

                if(branch == null) 
                {
                    return NotFoundError(branchName);
                }

                return Response(branch);
            }
        );
    }
}
```

After that modification GraphQL API returns exactly the same JSON like I showed at the beginnig of the story.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0002.gif)

Now we have to do similar things when we want to return collections. First we have to add additional GraphQL type which can store arrays in the `Data` property.

```cs
  public class ResponseListGraphType<TGraphType> : ObjectGraphType<Response> where TGraphType : GraphType
  {
      public ResponseListGraphType()
      {
          Name = $"ResponseList{typeof(TGraphType).Name}";

          Field(x => x.StatusCode, nullable: true).Description("Status code of the request.");
          Field(x => x.ErrorMessage, nullable: true).Description("Error message if requests fails.");

          Field<ListGraphType<TGraphType>>(
              "data",
              "Project data returned by query.",
              resolve: context => context.Source.Data
          );
      }
  }
```

And we can add new query and resolver for list.

```cs
public class BranchesResolver : Resolver, IBranchesResolver
{
    private readonly IBranchesService _branchesService;

    public BranchesResolver(IBranchesService branchesService)
    {
        _branchesService = branchesService;
    }

    public void Resolve(GraphQLQuery graphQLQuery)
    {
        graphQLQuery.Field<ResponseListGraphType<BranchType>>(
            "branches",
            arguments: new QueryArguments(
                new QueryArgument<NonNullGraphType<StringGraphType>> { Name = "projectId", Description = "id of the project" }
            ),
            resolve: context => { 
                var projectId = context.GetArgument<string>("projectId");
                var list = _branchesService.GetBranchesAsync(projectId).GetAwaiter().GetResult();

                return Response(list);
            }
        );

        graphQLQuery.Field<ResponseGraphType<BranchType>>(
            "branch",
            arguments: new QueryArguments(
                new QueryArgument<NonNullGraphType<StringGraphType>> { Name = "projectId", Description = "id of the project" },
                new QueryArgument<NonNullGraphType<StringGraphType>> { Name = "branchName", Description = "name of the branch" }
            ),
            resolve: context => { 
                var projectId = context.GetArgument<string>("projectId");
                var branchName = context.GetArgument<string>("branchName");
                var branch = _branchesService.GetBranchAsync(projectId, branchName).GetAwaiter().GetResult();

                if(branch == null) 
                {
                    return NotFoundError(branchName);
                }

                return Response(branch);
            }
        );
    }
}
```

Now all my request from GrapQL API is boxed with general information about each query status. Client applications can check that status and react accordingly.

---

If you are interested how I used GraphQL in .NET (Core) you can view [source code](https://github.com/BibliothecaTeam/Bibliotheca.Server.Gateway/tree/master/src/Bibliotheca.Server.Gateway.Api) of one of my project on GitHub.