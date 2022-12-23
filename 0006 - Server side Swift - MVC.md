# Server side Swift - MVC

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0015.jpeg)

> This is the article created at Feb 13, 2018 and moved from Medium.

I’m full stack developer. Most of my server side applications was written in .NET (ASP.NET Web API, ASP.NET MVC). However I spent a few last weeks on my side project Vcoin. It’s really simple iOS application written in Swift. I thought it would be great to invest my time to figure out how to create also server side application in Swift. I’ve read some great articles and introduction to that topic. Some of them you can find below:
<!--more-->

- [Benchmarks for the Top Server-Side Swift Frameworks vs. Node.js](https://medium.com/@rymcol/benchmarks-for-the-top-server-side-swift-frameworks-vs-node-js-24460cfe0beb)
- [Current Features & Benefits of the Top Server-Side Swift Frameworks](https://medium.com/@rymcol/current-features-benefits-of-the-top-server-side-swift-frameworks-b15b4f2d7bc3)
- [Server-side Swift Frameworks Comparison](https://www.netguru.co/codestories/server-side-swift-frameworks-comparison)

After reading these articles I decided to choose Perfect which is fast and jam-packed with features. However, I have not found anywhere example how to build server side production application in Swift/Perfect. How to divide application to some layers. Where to put business logic, validations, models, data access etc. Thus I decided to build some simple application based on my experience with ASP.NET Core MVC. In this article I’m focusing on MVC pattern. In next articles I will introduce data access (with some ORM framework), validation, authorization etc.

---

The basic idea behind MVC is to separate internal data models from the user interface via the controller and view.

MVC pattern in server side API:

- **M**odel — entities whcih should be persisted in database
- **V**iew — this layer in API is not so obvious there is no HTML/Storyboard files with view which be presented to the user, however user will see somethig else — JSON produced by our API (Controllers) and for me that JSON is View layer
- **C**ontroller — part with business logic, of course it would be great to have light controllers and move real business logic to compomenents that can be more reusable than controllers (like services).

---

### Model

My model is simple. I have two classes which stores information about tasks and users.

```swift
import Foundation

class Task : Codable {
    
    var id: Int
    var name: String
    var isFinished: Bool
    
    init(id: Int, name: String, isFinished: Bool) {
        self.id = id
        self.name = name
        self.isFinished = isFinished
    }
}
```

```swift
import Foundation

class User : Codable {
    
    var id: Int
    var name: String
    var email: String
    var isLocked: Bool
    
    init(id: Int, name: String, email: String, isLocked: Bool) {
        self.id = id
        self.name = name
        self.email = email
        self.isLocked = isLocked
    }
}
```

Later in the text I will focus only on `Tasks` model/controller.

---

### Controller

First I created base class for all my controllers.

```swift
import Foundation
import PerfectHTTP

class Controller {
    var routes = Routes()
    
    init() {
        initRoutes()
    }
    
    func initRoutes() {
    }
}
```

It’s really simple class. Main purpose of that class is to store routing which each controller is supposed to handle. I can create now my first controller.

```swift
import Foundation
import PerfectHTTP

class TasksController : Controller {
    
    private let tasksRepository: TasksRepositoryProtocol!
    
    init(tasksRepository: TasksRepositoryProtocol) {
        self.tasksRepository = tasksRepository
    }
    
    override func initRoutes() {
        routes.add(method: .get, uri: "/tasks", handler: getTasks)
        routes.add(method: .get, uri: "/tasks/{id}", handler: getTask)
        routes.add(method: .post, uri: "/tasks", handler: postTask)
        routes.add(method: .put, uri: "/tasks/{id}", handler: putTask)
        routes.add(method: .delete, uri: "/tasks/{id}", handler: deleteTask)
    }
    
    private func getTasks(request: HTTPRequest, response: HTTPResponse) {
        let tasks = self.tasksRepository.getTasks()
        response.sendJson(tasks)
    }
    
    private func getTask(request: HTTPRequest, response: HTTPResponse) {
        
        if let stringId = request.urlVariables["id"], let id = Int(stringId) {
            if let task = self.tasksRepository.getTask(id: id) {
                return response.sendJson(task)
            }
        }
        
        response.sendNotFound()
    }
    
    private func postTask(request: HTTPRequest, response: HTTPResponse) {
        do {
            let task = try request.getObjectFromRequest(Task.self)
            self.tasksRepository.addTask(task: task)
            
            return response.sendJson(task)
        }
        catch {
            response.sendBadRequest()
        }
    }
    
    private func putTask(request: HTTPRequest, response: HTTPResponse) {
        do {
            let task = try request.getObjectFromRequest(Task.self)
            self.tasksRepository.updateTask(task: task)
            
            return response.sendJson(task)
        }
        catch {
            response.sendBadRequest()
        }
    }
    
    private func deleteTask(request: HTTPRequest, response: HTTPResponse) {
        if let stringId = request.urlVariables["id"], let id = Int(stringId) {
            self.tasksRepository.deleteTask(id: id)
            return response.sendOk();
        }
        
        response.sendNotFound()
    }
}
```

I have two main concepts here:

- _Dependency Injection_ — I injected dependencies by constructor, `TaskRepositoryProtocol` is a simple protocol witch describe access to the model.
- _Routing Initializations_ — unfortunatelly Swift doesn’t support custom attributes and we cannot mark our methods like in ASP.NET MVC, that’s why I introduced method `initRoutes` where we have to connect routing to specific methods in controller.

Protocol which I injected to the controller is pretty simple.

```swift
protocol TasksRepositoryProtocol {
    func getTasks() -> [Task]
    func getTask(id: Int) -> Task?
    func addTask(task: Task)
    func updateTask(task: Task)
    func deleteTask(id: Int)
}
```

Implementation of that protocol is naive and operates on hardcoded objects.

```swift
class TasksRepository : TasksRepositoryProtocol {
    
    var tasks = [
        1: Task(id: 1, name: "Create new Perfect server", isFinished: false),
        2: Task(id: 2, name: "Improve MVC pattern on server side", isFinished: false),
        3: Task(id: 3, name: "Finish code refactoring", isFinished: false),
        4: Task(id: 4, name: "Move to newest fremeworks", isFinished: false)
    ]
    
    func getTasks() -> [Task] {
        return Array(self.tasks.values)
    }
    
    func getTask(id: Int) -> Task? {
        let filteredTasks = tasks.values.filter { (task) -> Bool in
            return task.id == id
        }
        
        return filteredTasks.first
    }
    
    func addTask(task: Task) {
        self.tasks[task.id] = task
    }
    
    func updateTask(task: Task) {
        self.tasks[task.id] = task
    }
    
    func deleteTask(id: Int) {
        self.tasks.removeValue(forKey: id)
    }
}
```

I will replace that implementations by ORM in future articles.

---

### Dependency Injection

As a DI container I choose [Dip](https://github.com/AliSoftware/Dip) which met my requirements and his syntaxt is clear and consistent (it looks much better for me then [Swinject](https://github.com/Swinject/Swinject) syntaxt).

```swift
import Foundation
import Dip

extension DependencyContainer {
    func configure() {
        self.registerRepositories(container: self)
        self.registerControllers(container: self)
    }
    
    func resolveAllControllers() -> [Controller] {
        let controllers:[Controller] = [
            try! container.resolve() as HealthController,
            try! container.resolve() as TasksController,
            try! container.resolve() as UsersController
        ]
        
        return controllers
    }
    
    private func registerRepositories(container: DependencyContainer) {
        container.register { TasksRepository() as TasksRepositoryProtocol }
        container.register { UsersRepository() as UsersRepositoryProtocol }
    }
    
    private func registerControllers(container: DependencyContainer) {
        container.register { TasksController(tasksRepository: $0) }
        container.register { UsersController(usersRepository: $0) }
        container.register { HealthController() }
    }
}
```

In `DependencyContainer` extension I have two methods (`registerRepositories`, `registerControllers`) which are responsible for adding to the container information about existing repositories and controllers.

There is also one additional method here: `resolveAllControllers` which is prepared especially for routing initialization which is described below.

### Routing Initialization

Unfortunately Swift hasn’t have so sophisticated reflection/introspection like .NET and we cannot use it to register routing automatically (for example by getting all classes that inherit from `Controller` class and construct object in runtime dynamically).

Thats why we have to do this manually in `main.swift` file.

```swift
import PerfectHTTP
import PerfectHTTPServer
import PerfectLib
import Dip

// Configure dependency injection.
let container = DependencyContainer()
container.configure()

// Initialize all controllers.
let controllers = container.resolveAllControllers()

// Routes and handlers.
var routes = Routes()
routes.configure(basedOnControllers: controllers)


do {
    // Launch the HTTP server.
    try HTTPServer.launch(
        .server(name: "www.example.ca", port: 8181, routes: routes))
} catch {
    fatalError("\(error)") // fatal error launching one of the servers
}
```

My `main.swift` file is small and consitent. First I’m creating `DependencyContainer` and configure it (by executing method which was described in the previous chapter).

Then I have to create list with all my controllers and based on that list I can create routings by below extension.

```swift
import Foundation
import PerfectHTTP

extension Routes {
    func configure(basedOnControllers controllers: [Controller]) {
        for controller in controllers {
            routes.add(controller.routes)
        }
    }
}
```

**That is basically all!** My server side application is divided now and looks much better then in the begging.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0016.png)

Now when I would like to add a new controller I will have to create a new Swift file with controller implementation and in `DependencyContainer` extension register a new controller in the container and add it to the list in `resolveAllControllers` method. When Swift will introduce some more extensive way for introspection this second step should be removed.

---

Github repository with source code (branch: `mvc-pattern`): [mczachurski/TaskServerSwift](https://github.com/mczachurski/TaskServerSwift).