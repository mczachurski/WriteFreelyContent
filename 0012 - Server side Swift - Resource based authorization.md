# Server side Swift - Resource based authorization

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0030.jpeg)

> This is the article created at Mar 4, 2018 and moved from Medium.

My previous articles described how I realized MVC pattern in server side Swift HTTP application. Now we will add resource based authorization (authorization to specific resource/object) to our API.
<!--more-->

---

Authorization described in current article is very similar to resource based authorization implemented by Microsoft in ASP.NET Core. Here you can read more about it: [Resource-based authorization in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/authorization/resourcebased?tabs=aspnetcore2x). In my implementation I will also have similar artifacts:

- authorization **requirement** — this is simple class which defines which authorization we would like to run (which requirements must be fulfilled). Requirement can have additional properties (like name, amount etc.) which we can use in handler.
- authorization **handler** — handler is a class that utilize requirement, user and resource. In handler based on requirement’s properties we can verify if user has access to the resource.
- authorization **policy** — policy is a set of requirements.
- authorization **service** — service is a class which based on resource and requirement can execute appropriate handlers.

Below I will show how I combined above artifacts and how I implemented this pattern in my application.

### Requirements

If we would like to implement resource based authorization in our application first we have to create some requirements. Requirement class have to implement protocol: `AuthorizationRequirementProtocol`. Below is simple example of requirement.

```swift
public class SameAuthorRequirement : AuthorizationRequirementProtocol {
    public init() {
    }
}
```

We can have also properties in requirements. For example:

```swift
import Foundation

public enum Operations {
    case create
    case read
    case update
    case delete
}

public class OperationAuthorizationRequirement : AuthorizationRequirementProtocol {
    public let operation: Operations
    
    init(operation: Operations) {
        self.operation = operation
    }
}
```

Both of that requirements we will use in next chapters of the article.

### Authorization handlers

Authorization handlers are the most important things in resource based authorization. Here we have code which have to verify user access to the resource. Our handlers have to implement below protocol:

```swift
import Foundation
public protocol AuthorizationHandlerProtocol {
  
  var requirementType: AuthorizationRequirementProtocol.Type { get }
  var resourceType: EntityProtocol.Type { get }
  
  func handle(user: UserCredentials, resource: EntityProtocol, requirement: AuthorizationRequirementProtocol) throws -> Bool
}
```

We have three things here:

- `requirementType` — type of requirement which is consumed by handler
- `resourceType` — type of resource to which access will be checked
- `handle` — method wich will be executed during access verification

Below there are two examples of handler implementation. First based on `OperationAuthorizationRequirement` verify that user can read/update/delete task.

```swift
import Foundation

public class TaskOperationAuthorizationHandler : AuthorizationHandlerProtocol {
    
    public var requirementType: AuthorizationRequirementProtocol.Type   = OperationAuthorizationRequirement.self
    public var resourceType: EntityProtocol.Type                        = Task.self
    
    public func handle(user: UserCredentials, 
                       resource: EntityProtocol, 
                       requirement: AuthorizationRequirementProtocol) throws -> Bool {
        
        guard let task = resource as? Task else {
            throw AuthorizationTypeMismatchError()
        }
        
        if task.userId == user.id {
            return true
        }
        
        return false
    }
    
    public required init() {
    }
}
```

Second handler is very similar, it uses `SameAuthorRequrement` and it verifies if user is author of the task.

```swift
import Foundation

public class TaskAuthorizationHandler : AuthorizationHandlerProtocol {
    
    public var requirementType: AuthorizationRequirementProtocol.Type   = SameAuthorRequirement.self
    public var resourceType: EntityProtocol.Type                        = Task.self
    
    public func handle(user: UserCredentials, 
                       resource: EntityProtocol, 
                       requirement: AuthorizationRequirementProtocol) throws -> Bool {
        
        guard let task = resource as? Task else {
            throw AuthorizationTypeMismatchError()
        }
        
        if task.userId == user.id {
            return true
        }
        
        return false
    }
    
    public required init() {
    }
}
```

Implementation of both handlers are the same but now you should know the idea.

We have two possibilities how to use handlers:

- based on requirements,
- based on protocols.

### Authorization based on requirement

Here our `AuthorizationService` based on requirement and resource will directly invoke all handlers which was registered for the same types.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0031.png)

#### Registration

First thing which we have to do is adding our handler to the `AuthorizationService`:

```swift
// Resource-based authorization.
let authorizationService = try! container.resolve() as AuthorizationServiceProtocol
authorizationService.add(authorizationHandler: TaskOwnerAuthorizationHandler())
```

#### Authorization

Now we can use `AuthorizationService` in our application. For example we can check access to the task in our controller.

```swift
    public func get(request: HTTPRequest, response: HTTPResponse) {
        do {
            guard let stringId = request.urlVariables["id"], let id = UUID(uuidString: stringId) else {
                return response.sendBadRequestError()
            }
                
            guard let task = try self.tasksService.get(byId: id) else {
                return response.sendNotFoundError()
            }
            
            guard let user = request.getUserCredentials() else {
                return response.sendUnauthorizedError()
            }
            
            if try !self.authorizationService.authorize(user: user,
                    resource: task,
                    requirement: OperationAuthorizationRequirement(operation: .read)) {
                return response.sendForbiddenError()
            }
            
            let taskDto = TaskDto(task: task)
            return response.sendJson(taskDto)
        }
        catch {
            response.sendInternalServerError(error: error)
        }
    }
```

### Authorization based on policy

Here our `AuthorizationService` based on policy name will invoke all handlers which was registered for specific policy.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0032.png)

#### Registration

We have to add a new policy with all combined requirements.

```swift
// Resource-based authorization.
let authorizationService = try! container.resolve() as AuthorizationServiceProtocol
authorizationService.add(policy: "EditPolicy", requirements: [SameAuthorRequirement()])
```

#### Authorization

Now we can use new policy in our application. For example in controller class.

```swift
    public func get(request: HTTPRequest, response: HTTPResponse) {
        do {
            guard let stringId = request.urlVariables["id"], 
                  let id = UUID(uuidString: stringId) else {
                return response.sendBadRequestError()
            }
                
            guard let task = try self.tasksService.get(byId: id) else {
                return response.sendNotFoundError()
            }
            
            guard let user = request.getUserCredentials() else {
                return response.sendUnauthorizedError()
            }
                        
            if try !self.authorizationService.authorize(user: user, 
                                                        resource: task, 
                                                        policy: "EditPolicy") {
                return response.sendForbiddenError()
            }
            
            let taskDto = TaskDto(task: task)
            return response.sendJson(taskDto)
        }
        catch {
            response.sendInternalServerError(error: error)
        }
    }
```

---

Above examples were simple, however now we have all pieces in place to build much more complex authorization.

---

All source codes you can find in my GitHub project (branch `resource-authentication`): [mczachurski/TaskServerSwift](https://github.com/mczachurski/TaskServerSwift).