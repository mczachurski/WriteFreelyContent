# Server side Swift - Services, Repositories & Data validation

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0022.jpeg)

> This is the article created at Feb 26, 2018 and moved from Medium.

My previous articles described how I realized MVC pattern in server side Swift HTTP application, how I added support for configuration settings and ORM framework. Now we will add additional layers for separating database from our controllers. We will add repositories (for data access) and services (for business logic). Of course we have to be very careful and we should do only things that are necessary in our application. Generally we should build small applications (microservices) than one big/fat application (monolith). However even in microservices it’s good to introduce layers separation when we would like to have advantages like easier unit testing, reusable of business logic etc.
<!--more-->

---

In my server side applications I’m following one of two patterns:

- _repository pattern_
- _query/command pattern_

Below I will shortly describe both.

#### Repository pattern

Repository pattern is really simple. We encapsulate access to the database in a simple classes which shouldn’t have any complex logic. This is only naive abstraction layer to our database. Below there is a draw which ilustrate that pattern.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0023.png)

You can ask now:

> Ok, but now we have two data access layers: “Repositories” and “Data Access Layer (ORM)”.

Yes it’s partially true. But we have a few advantages:

- **encapsulation** — wrapping simple data access logic in methods. For example we can have method in repository: `getAvailablePlayers`. We can use that method in a lot of places in our business logic (services) layers. Now when we decide that we should change what that method should returns, we can change this in one place. If we will use directly ORM then we will have a lot of places when we should modify.
- **mocks** — we can easily create mocks for our repositories which is very often really hard if we will decide to use ORM directly.

Main disadvantage is that very often we have very similar code in our repository and business logic. If we would like to return data to our presentation layer from database we have to add method to repository interface (protocol) and implement it. Then we have to add similar method to the service interface (protocol) and implement it (very often we will write only single line of code — call repository).

#### Command/Query pattern

CQRS is very powerful pattern. We can separate read and write to the database. Thanks to this we can save data to one persistent store (for example database) and read from another. Especially when we would like to improve read performance. We can have different schemas in both databases and prepare data in that way that reading will be very fast (e.g. by denormalization).

Below there is a draw which illustrate that pattern.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0024.png)

In that pattern we don’t duplicate the code. And we have other advantages. We are sending events which can be handle by multiple handlers. We can easily log all events in our system and we can easily add new handlers when we would like to change some behavior.

We can mix both patterns and we can add to the CQRS additional layer (repositories) instead of using directly ORM (data access layer). This will be useful when we cannot create mocks for our ORM.

Of course in small applications there is no sense to implement so complex pattern. That’s why below I will describe how to implement repository pattern (with services and data validation) to our server side Swift application.

---

### Repositories

Becouse our repositories will have common methods (`get`, `add`, `update`, `delete`) we can abstract that methods to some base class. But first we have to add protocol that all our business entities will implement (our base repository class will operate on that protocol).

Here is our protocol `EntityProtocol`:

```swift
import Foundation

public protocol EntityProtocol : Codable {
  var id: Int { get set }
}
```

Our business entities should implement that protocol:

```swift
import Foundation

public class Task : EntityProtocol {
  public var id: Int
  public var name: String
  public var isFinished: Bool
  init(id: Int, name: String, isFinished: Bool) {
    self.id = id
    self.name = name
    self.isFinished = isFinished
  }
}
```

Now we can introduce `BaseRepository`:

```swift
import Foundation
import PerfectCRUD

public class BaseRepository<T: EntityProtocol> {
  internal let databaseContext: DatabaseContextProtocol
  
  init(databaseContext: DatabaseContextProtocol) {
    self.databaseContext = databaseContext
  }

  func get() throws -> [T] {
    let tasks = try self.databaseContext.set(T.self).select()
    return tasks.sorted { 
      (entity1, entity2) -> Bool in return entity1.id < entity2.id 
    }
  }

  func get(byId id: Int) throws -> T? {
    let task = try self.databaseContext.set(T.self)
        .where(\T.id == id).first()
    return task
  }

  func add(entity: T) throws {
    try self.databaseContext.set(T.self).insert(entity)
  }

  func update(entity: T) throws {
    try self.databaseContext.set(T.self)
        .where(\T.id == entity.id).update(entity)
  }

  func delete(entityWithId id: Int) throws {
    try self.databaseContext.set(T.self).where(\T.id == id).delete()
  }
}
```

As you can see `BaseRepository` is generic class which has implemented main CRUD operations.

Now we can create our first repository. For example repository for `Task` entity can look like this:

```swift
import Foundation
import PerfectCRUD

public protocol TasksRepositoryProtocol {  
  func get() throws -> [Task]
  func get(byId id: Int) throws -> Task?
  func add(entity: Task) throws
  func update(entity: Task) throws
  func delete(entityWithId: Int) throws
}

class TasksRepository : 
  BaseRepository<Task>, TasksRepositoryProtocol {
}
```

We have two things here:

- `TasksRepositoryProtocol` — protocol which we will use in all places in our application. We will inject it to our services and mock in our unit tests.
- `TasksRepository` — real implementation of tasks repository (all methods are implemented in `BaseRepository<Task>`.). In the application we should use that class directly only when we are adding registration to our _Dependency Container_.

Very similar thing we can do in our unit tests. So we can introduce `FakeBaseRepository`. This class can looks like below:

```swift
import Foundation
import TaskerServerLib
import Dobby

class FakeBaseRepository<T: EntityProtocol> {
  let getMock = Mock<()>()
  let getStub = Stub<(), [T]>()
  let getByIdMock = Mock<Int>()
  let getByIdStub = Stub<Int, T?>()
  let addMock = Mock<T>()
  let updateMock = Mock<T>()
  let deleteMock = Mock<Int>()

  func get() -> [T] {
    getMock.record(())
    return try! getStub.invoke(())
  }

  func get(byId id: Int) -> T? {
    getByIdMock.record(id)
    return try! getByIdStub.invoke((id))
  }

  func add(entity: T) {
    addMock.record(entity)
  }

  func update(entity: T) {
    updateMock.record(entity)
  }

  func delete(entityWithId id: Int) {
    deleteMock.record(id)
  }
}
```

And our fake repository will be really simple.

```swift
import Foundation
import TaskerServerLib

class FakeTasksRepository : 
  FakeBaseRepository<Task>, TasksRepositoryProtocol {
}
```

Our additional data access layer is ready. We have simple repository which we can use in services and easly mock in our unit tests. We have place when we can add additional methods for data access.

---

### Services

Now it’s time to introduce services. Place where we will have our main business logic. Why we are creating that classes? Some adventures:

- reusable business logic — instead of having everything in cotroller classes we have separate classes with methods which we can be easily use by other components then our controllers.
- thin controllers — becouse we are moving from controllers a lot of code to services we will have thin controllers. You can read about thin/fat controllers in [here](http://gunnarpeipman.com/2015/05/why-to-avoid-fat-controllers/) and [here](https://jonhilton.net/2016/05/23/3-ways-to-keep-your-asp-net-mvc-controllers-thin/).

Here we also have protocol and implementation (similar like for repositories).

```swift
import Foundation

public protocol TasksServiceProtocol {
  func get() throws -> [Task]
  func get(byId id: Int) throws -> Task?
  func add(entity: Task) throws
  func update(entity: Task) throws
  func delete(entityWithId id: Int) throws
}

public class TasksService : TasksServiceProtocol {
  private let tasksRepository: TasksRepositoryProtocol
  
  init(tasksRepository: TasksRepositoryProtocol) {
    self.tasksRepository = tasksRepository
  }

  public func get() throws -> [Task] {
    return try self.tasksRepository.get()
  }

  public func get(byId id: Int) throws -> Task? {
    return try self.tasksRepository.get(byId: id)
  }

  public func add(entity: Task) throws {
    if !entity.isValid() {
      let errors = entity.getValidationErrors()
      throw ValidationsError(errors: errors)
    }
    try self.tasksRepository.add(entity: entity)
  }

  public func update(entity: Task) throws {
    if !entity.isValid() {
      let errors = entity.getValidationErrors()
      throw ValidationsError(errors: errors)
    }
    try self.tasksRepository.update(entity: entity)
  }

  public func delete(entityWithId id: Int) throws {
    try self.tasksRepository.delete(entityWithId: id)
  }
}
```

We will use protocols in our application (we will inject them to the controllers for example).

Above service is really simple. We are generally wrapping out our repository. There is only one thing added here: _validation_. For example in method `add(entity: Task)` we have code:

```swift
if !entity.isValid() {
  let errors = entity.getValidationErrors()
  throw ValidationsError(errors: errors)
}

try self.tasksRepository.add(entity: entity)
```

Thus only if our entity is valid we will add it to the database. In other case we will throw validation error. Below I described how it is realized.

---

### Data validations

I created really simple data validation. First I’ve added Swift error which will store information about errors in model.

```swift
import Foundation

public class ValidationsError : Error {
  let errors: [String: String]
  init(errors: [String: String]) {
    self.errors = errors
  }
}
```

Then I added extension to my class (in separate file). Here I have two methods: `isValid()` and `getValidationErrros()`. Below there is simple example for `Task` class.

```swift
import Foundation

public extension Task {
  
  public func isValid() -> Bool {
    return !self.name.isEmpty
  }
  public func getValidationErrors() -> [String: String] {
    var errors: [String: String] = [:]
    if self.name.isEmpty {
      errors["name"] = "Field is required."
    }
    return errors
  }
}
```

Now I can use that methods in my service (example was shown in previous chapter). I done this in service instead of controller, becouse now if I will use service somewhere else my validation will be also executed.

Now I have add **_catch_** to my controller.

```swift
public func postTask(request: HTTPRequest, response: HTTPResponse) {
  do {
    let task = try request.getObjectFromRequest(Task.self)
    try self.tasksService.add(entity: task)
    return response.sendJson(task)
  }
  catch let error as ValidationsError {
    response.sendValidationsError(error: error)
  }
  catch {
    response.sendInternalServerError(error: error)
  }
}
```

Thus when user send to controller incorrect data, controller will catch all validation errors and send to him information about them. He should got in response JSON (and status code _400 Bad request_):

```json
{
  "message" : "Error during entity validation.",
  "errors" : {
    "name" : "Field is required."
  }
}
```

---

Skeleton of my application is almost ready. We have controllers, services, repositories and ORM. We can read settings from configuration files and whole our code is tested by unit tests.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0025.png)

In my next article I will add support for authentication and authorization.

---

All source codes you can find in my GitHub project (branch `repositories`): [mczachurski/TaskServerSwift](https://github.com/mczachurski/TaskServerSwift).