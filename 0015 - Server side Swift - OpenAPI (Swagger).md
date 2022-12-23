# Server side Swift - OpenAPI (Swagger)

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0048.png)

> This is the article created at Apr 3, 2018 and moved from Medium.

REST API is very popular and common way to expose APIs to software applications. However, besides undeniable advantages it also has disadvantages. One of them is lack of common interface for retrieving information about API endpoints (list of endpoints, request/response object’s metadata, possible status codes, authorization etc.).
<!--more-->

---

Of course nature abhors a vacuum and developers create frameworks and specifications which are very helpful here. Most common solution was created by [OpenAPI Initiative](https://www.openapis.org). [Here](https://github.com/OAI/OpenAPI-Specification) you can find detailed description of OpenAPI specification. We also have great tools which can present API in readable for humans format:

- Swagger UI: [https://swagger.io/swagger-ui/](https://swagger.io/swagger-ui/)
- OpenAPI 3 Viewer: [https://github.com/koumoul-dev/openapi-viewer](https://github.com/koumoul-dev/openapi-viewer)
- Lincoln: [https://github.com/temando/open-api-renderer](https://github.com/temando/open-api-renderer)
- Widdershins: [https://github.com/Mermade/widdershins](https://github.com/Mermade/widdershins)

Above tools are very helpful during developing and testing APIs, especially when you are working in a team and testers aren’t technical people and/or they cannot read the source code. Of course you can use that specification also for generating documentation for other people like product owners, technical writers or customers.

The question now arises: how to use OpenAPI specification in Swift projects? I tried to found some framework or other way to achieve that, but I failed. I didn’t found any solution that I would be happy with. For .NET projects there is for example great project: [Swashbuckle](https://github.com/domaindrivendev/Swashbuckle) (and [Swashbucle.AspNetCore](https://github.com/domaindrivendev/Swashbuckle.AspNetCore)). It uses reflection for generating OpenAPI specification and I would like to have something similar for Swift.

---

I decided to create my own simple framework: [Swiftgger](https://github.com/mczachurski/Swiftgger) and use it in my server side (example) Swift project. On github page you can find detailed description how to use it in your project. Unfortunately it’s not so automatic like Swashbuckle — because Swift hasn’t have so extended mechanism of reflection/introspection. Basically you have to do four things:

**1. create a builder** — here you can specify basic information about API (title, version, authors, authorization etc.)

```swift
let openAPIBuilder = OpenAPIBuilder(
    title: "Tasker server API",
    version: "1.0.0",
    description: "This is a description."
)
`

**2. add information about controllers** — basic information about controller like name and description, it also contains list of actions included in controller

```swift
openAPIBuilder.addController(
    APIController(name: "Users", 
                  description: "Controller for users", 
                  actions: []
    )
)
```

**3. add information about actions** — this is the most important part, here we have specify detailed description of each action like route, response and request types, status codes etc.

```swift
APIAction(method: .get, route: "/users/{id}",
    summary: "Getting user by id",
    description: "Action for getting specific user from server",
    parameters: [
        APIParameter(name: "id", description: "User id", required: true)
    ],
    responses: [
        APIResponse(code: "200", description: "Specific user", object: UserDto.self),
        APIResponse(code: "404", description: "User with entered id not exists"),
        APIResponse(code: "401", description: "User not authorized")
    ],
    authorization: true
)
```

**4. add information about schema (model)** — model contains information about each object which can be used by your API (in my case I have to put here all my DTOs objects). Here we have instances of objects, because values will be shown in Swagger as an examples.

```swift
openAPIBuilder.addObjects([
    APIObject(object: UserDto(id: UUID(), createDate: Date(), name: "John Doe", email: "email@test.com", isLocked: false)),
    APIObject(object: ValidationErrorResponseDto(message: "Object is invalid", errors: ["property": "Information about error."]))
])
```

This is minimal steps which are required. However Swiftgger provides other methods where you can define type of authorization to your API (Basic/Bearer authorization), server’s addresses, licenses etc.

When you have all above steps already done you have to execute one method:

```swift
let document = try openAPIBuilder.build()
```

This will create document which fulfill [OpenAPI 3.0 specification](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.1.md). Now you have to expose that object as a JSON and use it in [Swagger UI](https://swagger.io/swagger-ui/).

---

API to my application (TaskServer application) has great description which can be used by my team:

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0049.png)

Here you can find JSON which is generated by Swiftgger based on definition done in TaskServer application: [https://taskerswift.azurewebsites.net/openapi](https://taskerswift.azurewebsites.net/openapi).

And here there is a Swagger UI generated based on that above JSON response: [https://taskerswift-swagger.azurewebsites.net/](https://taskerswift-swagger.azurewebsites.net/).

---

Of course I would like to have my API description always current. That’s why I created two protocols:

**Controller protocol**

```swift
public protocol ControllerProtocol {
    func getActions() -> [ActionProtocol]    
    func getMetadataDescription() -> String    
    func getMetadataName() -> String
}
```

**Action protocol**

```swift
public protocol ActionProtocol {     
    func getHttpMethod() -> HTTPMethod    
    func getUri() -> String    
    func handler(request: HTTPRequest, response: HTTPResponse)     
    func getMetadataSummary() -> String    
    func getMetadataDescription() -> String    
    func getMetadataAuthorization() -> AuthorizationPolicy     
    func getMetadataParameters() -> [APIParameter]?    
    func getMetadataRequest() -> APIRequest?    
    func getMetadataResponses() -> [APIResponse]?
}
```

All my controllers and actions have to implement above protocols and I’m generating OpenAPI schema based on that information. Thus I have description of controllers/actions near real handlers implementations. It’s very helpful for keeping the description consistent with the implementation.

---

You can find full example [here](https://github.com/mczachurski/TaskServerSwift) on my example Github project. If you have some questions/problems with Swiftgger please use [issues](https://github.com/mczachurski/Swiftgger/issues) on Github. Of course all “pull requests” all welcome.