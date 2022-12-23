# Server side Swift - MVC (unit tests)

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0017.jpeg)

> This is the article created at Feb 17, 2018 and moved from Medium.

In my [previous article](https://writefreely.social/mczachurski/server-side-swift-mvc) I described how to build server side Swift application according to MVC pattern (with construction similar to ASP.NET MVC). However there was missing one important thing: **unit tests**. I cannot imagine serious application without unit tests. In this article I’m describing how to add project with unit tests to our MVC application in Swift (and Perfect framework).
<!--more-->

---

First we have to rebuild a little bit our application. We cannot create unit tests for executable application. That is why we have to divide our application into two parts:

- **executable** — this part should contain as few files as possible,
- **library** — that part should contains whole business logic — and this part we can connect to our test library.

After dividing application we should have project which looks like below:

```
Sources/
    TaskerServerApp/
        main.swift
    TaskerServerLib/
        Controllers/*
        Models/*
        Repositories/*
        Extensions/*
Tests
    TaskerServerLibTests/
```

Thus we have `TaskerServerApp` executable which contains only one file (main file which bootstrap our server). We have `TaskerServerLib` which contains main part of our business logic (controllers, model, repositories etc.). In separate project `TaskerServerLibTests` we will have our unit tests.

To create that project we have to prepate `Package.swift` file for _Swift Package Manager_. It should looks like that:

```swift
// swift-tools-version:4.0
// The swift-tools-version declares the minimum version of Swift required to build this package.

import PackageDescription

let package = Package(
    name: "TaskerServer",
    dependencies: [
        // Dependencies declare other packages that this package depends on.
        .package(url: "https://github.com/PerfectlySoft/Perfect-HTTPServer.git", from: "3.0.0"),
        .package(url: "https://github.com/AliSoftware/Dip", from: "6.0.0"),
        .package(url: "https://github.com/mczachurski/Dobby", from: "0.7.1")
    ],
    targets: [
        // Targets are the basic building blocks of a package. A target can define a module or a test suite.
        // Targets can depend on other targets in this package, and on products in packages which this package depends on.
        .target(name: "TaskerServerApp", dependencies: ["TaskerServerLib", "PerfectHTTPServer", "Dip"]),
        .target(name: "TaskerServerLib", dependencies: ["PerfectHTTPServer", "Dip"]),
        .testTarget(name: "TaskerServerLibTests", dependencies: ["TaskerServerLib", "PerfectHTTPServer", "Dip", "Dobby"])
    ]
)
```

Besides three targets definition I have three packages dependencies:

- Perfect-HTTPServer — web server and toolkit,
- Dip — dependency injection container,
- Dobby — helpers for mocking and stubbing.

After moving all files to appropriates folders we can execute commands:

```sh
$ swift build  
$ swift package generate-xcodeproj
```

_Swift Package Manager_ should download dependencies, build project and generate project file for Xcode. Now we can run `swift run` command to run our HTTP server or open project in Xcode.

---

After dividing our executable we can create first unit tests. First I would like to create unit tests which will check routing. Example of that test is below:

```swift
import XCTest
import PerfectHTTP
import Dobby
@testable import TaskerServerLib

class UsersControllerTests: XCTestCase {
    
    func testInitRoutesShouldInitializeGetAllUsersRoute() {
        
        // Arrange.
        let fakeHttpRequest = FakeHTTPRequest(method: .get)
        let fakeUsersRepository = FakeUsersRepository()
        
        // Act.
        let usersController = UsersController(usersRepository: fakeUsersRepository)
        
        // Assert.
        let requestHandler = usersController.routes.navigator.findHandler(uri: "/users", webRequest: fakeHttpRequest)
        XCTAssertNotNil(requestHandler)
    }
}
```

As you can see in `Arrange` section we are creating fake objects (which are implementing appropriate protocols). In `Act` section we are creating our controller (in constructor there is routing initialization). And in `Assert` section we are validating if there is defined any handler for specific `uri`.

We can run unit tests in Xcode or by executing `swift test` command.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0018.png)

Now we have to consider how to create fake implementations. In languages like C#/Java we can use powerfull mocking frameworks like: [Moq](https://github.com/moq/moq4), [Rhino.Mocks](https://github.com/ayende/rhino-mocks), [mockito](https://github.com/mockito/mockito). Unfortunately in Swift this task is not so easy. Swift doesn’t have good enough reflection/introspection which can help with generating dynamic classes (in many mocking frameworks known as proxies) in runtime. I found a few mocking libraries like:

[SwiftyMocky](https://github.com/MakeAWishFoundation/SwiftyMocky) — we can mark our protocols (`//sourcery:AutoMockable`) and appropiate fake implementation will be generated by [Sourcery](https://github.com/krzysztofzablocki/Sourcery) tool,

[Cuckoo](https://github.com/Brightify/Cuckoo) — special generator (`CuckooGenerator`) and framework, generator is responsible for generating fake implementations based on configuration file,

[Mimus](https://github.com/AirHelp/Mimus) — framework which deliver methods which we can use in our manually generated fake files (currently only mocks are supported),

[Dobby](https://github.com/trivago/Dobby) — similar to Mimus, however in our fake implementations we can use mocks and stubs.

I do not want to use any generators, because in big distributed teams (especially when team members can use different operating system) it will generate many issues. I decided to choose Dobby framework which is really simple and provide clear and consistent syntax. Of course still we will have to manually create our fake implementation files which sometimes can be painful.

Below is example how to create fake repository implementation:

```swift
import Foundation
import TaskerServerLib
import Dobby

class FakeUsersRepository : UsersRepositoryProtocol {
    
    let getUsersMock = Mock<()>()
    let getUsersStub = Stub<(), [User]>()

    let getUserMock = Mock<(Int)>()
    let getUserStub = Stub<(Int), User?>()

    let addUserMock = Mock<(User)>()
    let updateUserMock = Mock<(User)>()
    let deleteUserMock = Mock<(Int)>()
        
    func getUsers() -> [User] {
        getUsersMock.record(())
        return try! getUsersStub.invoke(())
    }
    
    func getUser(id: Int) -> User? {
        getUserMock.record(id)
        return try! getUserStub.invoke((id))
    }
    
    func addUser(user: User) {
        addUserMock.record(user)
    }
    
    func updateUser(user: User) {
        updateUserMock.record(user)
    }
    
    func deleteUser(id: Int) {
        deleteUserMock.record(id)
    }
}
```

Thanks to that mock file we can write tests that look like this:

```swift
func testGetUsersShouldReturnUsersCollection() {

    // Arrange.
    let fakeHttpRequest = FakeHTTPRequest()
    let fakeHttpResponse = FakeHTTPResponse()
    let fakeUsersRepository = FakeUsersRepository()

    fakeUsersRepository.getUsersMock.expect(any())
    fakeUsersRepository.getUsersStub.on(any(), return: [
            User(id: 1, name: "John Doe", email: "john.doe@emailx.com", isLocked: false),
            User(id: 2, name: "Victor Doe", email: "victor.doe@emailx.com", isLocked: false)
    ])
    let usersController = UsersController(usersRepository: fakeUsersRepository)

    // Act.
    usersController.getUsers(request: fakeHttpRequest, response: fakeHttpResponse)

    // Assert.
    let users = try! fakeHttpResponse.getObjectFromResponseBody(Array<User>.self)
    fakeUsersRepository.getUsersMock.verify()
    XCTAssertEqual(HTTPResponseStatus.ok.code, fakeHttpResponse.status.code)
    XCTAssertEqual(2, users.count)
    XCTAssertEqual("john.doe@emailx.com", users[0].email)
    XCTAssertEqual("victor.doe@emailx.com", users[1].email)
}

func testGetUserShouldReturnUserWhenWeProvideCorrectId() {

    // Arrange.
    let fakeHttpRequest = FakeHTTPRequest(urlVariables: ["id": "1"])
    let fakeHttpResponse = FakeHTTPResponse()
    let fakeUsersRepository = FakeUsersRepository()

    fakeUsersRepository.getUserMock.expect(any())
    fakeUsersRepository.getUserStub.on(equals(1), return:
        User(id: 1, name: "John Doe", email: "john.doe@emailx.com", isLocked: false)
    )
    let usersController = UsersController(usersRepository: fakeUsersRepository)

    // Act.
    usersController.getUser(request: fakeHttpRequest, response: fakeHttpResponse)

    // Assert.
    let users = try! fakeHttpResponse.getObjectFromResponseBody(User.self)
    fakeUsersRepository.getUserMock.verify()
    XCTAssertEqual(HTTPResponseStatus.ok.code, fakeHttpResponse.status.code)
    XCTAssertEqual(1, users.id)
    XCTAssertEqual("john.doe@emailx.com", users.email)
}

func testGetUserShouldReturnNotFoundStatusCodeWhenWeProvideIncorrectId() {

    // Arrange.
    let fakeHttpRequest = FakeHTTPRequest(urlVariables: ["id": "2"])
    let fakeHttpResponse = FakeHTTPResponse()
    let fakeUsersRepository = FakeUsersRepository()

    fakeUsersRepository.getUserMock.expect(any())
    fakeUsersRepository.getUserStub.on(equals(2), return: nil)
    let usersController = UsersController(usersRepository: fakeUsersRepository)

    // Act.
    usersController.getUser(request: fakeHttpRequest, response: fakeHttpResponse)

    // Assert.
    XCTAssertEqual(HTTPResponseStatus.notFound.code, fakeHttpResponse.status.code)
}
```

As you can see our tests are simple and clear. We have one common way to create stubs/mocks. There is much more unit tests in my example project (link is at the end).

---

After all above changes our server side project looks like on below screen.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0019.png)

We have three targets (executable, library with business logic and library with unit tests). We have clear separation between server bootstrap (`main.swift` file), controllers (routings) and model. And what is important we have **unit tests**!

---

In next article I will focus on next important topic: _configuration files_. Files which can store information important for our server like: connection strings, other services addresses, logs etc.

---

All source codes you can find in my GitHub project (branch `unit-tests`): [mczachurski/TaskServerSwift](https://github.com/mczachurski/TaskServerSwift).