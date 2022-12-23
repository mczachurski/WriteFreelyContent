# Server side Swift - Configuration files

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0020.jpeg)

> This is the article created at Feb 18, 2018 and moved from Medium.

In my two previous articles I described how I added MVC pattern (with unit tests) to server side Swift HTTP server. In this article I will show how I added support for configuration files to the application.
<!--more-->

#### Configuration framework

In ASP.NET Core MVC application Microsoft added support for reading configuration files (and also environment variables, command parameters etc.) and that syntaxt and idea looks pretty cool. Below there is sample of code which can be used to read configuration in Microsoft framework.

```cs
var builder = new ConfigurationBuilder()
    .SetBasePath(env.ContentRootPath)
    .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
    .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true)
    .AddEnvironmentVariables();

Configuration = builder.Build();
```

I would like to achieve similar construction in my Swift application. It is really important that we can decide which parameters are most important (based on order of executed method). I would like to have order: `file -> system environment -> command parameters`. First I thought that I will have to create that classes myself but then I found really good library created by IBM: [IBM-Swift/Configuration](https://github.com/IBM-Swift/Configuration).

This is really good framework it fulfills all my requirements. And it’s very similar to Microsoft solution.

#### Read configurations

To read configurations from all places which I would like to support we have to add to our `main.swift` file the following code.

```swift
// Read configuration file.
let configurationManager = ConfigurationManager()
    .load(file: "configuration.json", relativeFrom: .pwd)
    .load(.environmentVariables)
    .load(.commandLineArguments)

let configuration = configurationManager.build()
```

It is important to specify `BasePath.pwd` as a `relativeFrom` parameter. It is relative path from present working directory (`PWD`).

My whole `main.swift` file now looks like this:

```swift
import PerfectHTTP
import PerfectHTTPServer
import PerfectLib
import Dip
import TaskerServerLib
import Configuration

// Read configuration file.
let configurationManager = ConfigurationManager()
    .load(file: "configuration.json", relativeFrom: .pwd)
    .load(.environmentVariables)
    .load(.commandLineArguments)

let configuration = configurationManager.build()

// Configure dependency injection.
let container = DependencyContainer()
container.configure(withConfiguration: configuration)

// Initialize all controllers.
let controllers = container.resolveAllControllers()

// Routes and handlers.
var routes = Routes()
routes.configure(basedOnControllers: controllers)


do {
    // Launch the HTTP server.
    try HTTPServer.launch(
        .server(name: configuration.serverName, port: configuration.serverPort, routes: routes))
} catch {
    fatalError("\(error)") // fatal error launching one of the servers
}
```

After reading configuration by using IBM’s framework I’m transcripting configuration settings to my own object — which stores as a properties only configuration that I’m using in the application. Thanks to this I will use real class properties in my application instead of string keys. I will have intellisense and much easier refactoring. For that purposes I created simple extension to `ConfigurationManager` class. Which only creates `Configuratio` object based on properties from manager.

```swift
import Foundation
import Configuration

extension ConfigurationManager {
    
    public func build() -> Configuration {
        let configuration = Configuration(manager: self)
        return configuration
    }
}
```

My `Configuration` object is really simple. It stores all configuration properties which I can use in whole application.

```swift
import Foundation
import Configuration

public class Configuration {
    public var serverName: String
    public var serverPort: Int
    public var logFile: String
    public var databaseHost: String
    public var datanaseName: String
    public var databaseUser: String
    public var databasePassword: String
    
    init(manager: ConfigurationManager) {
        self.serverName = manager["serverName"] as? String ?? ""
        self.serverPort = manager["serverPort"] as? Int ?? 0
        self.logFile = manager["logFile"] as? String ?? ""
        self.databaseHost = manager["databaseHost"] as? String ?? ""
        self.datanaseName = manager["datanaseName"] as? String ?? ""
        self.databaseUser = manager["databaseUser"] as? String ?? ""
        self.databasePassword = manager["databasePassword"] as? String ?? ""
    }
}
```

My `configuration.json` corresponds to the above class and looks like this:

```json
{
    "serverName": "www.example.com",
    "serverPort": 8181,
    "logFile": "logs/log.txt",
    "databaseHost": "localhost",
    "datanaseName": "tasker",
    "databaseUser": "tasker_user",
    "databasePassword": "P@ssw0rd!"
}
```

We have to put that file near `Package.swift` file. This is important when we are working in Xcode because this folder is current working folder and `ConfigurationManager` will correctly locate this file. When we have rebuilded executable file then our configuration file have to be placed near that file.

#### Dependency injection

Now I have all my configuration stored in created specially for that purpose object. Now I would like to have method to use that configuration in all places that need access to configuration settings. Of course the best solution for that is dependency injection. I will register my configuration object in my dependency container and the I can resolve configuration wherever I would like. Registering can be done as into following code.

```swift
//
//  DependencyContainer.swift
//  TaskerServerPackageDescription
//
//  Created by Marcin Czachurski on 12.02.2018.
//

import Foundation
import Dip

extension DependencyContainer {
    public func configure(withConfiguration configuration: Configuration) {
        self.registerConfiguration(container: self, configuration: configuration)
        self.registerRepositories(container: self)
    }
    
    private func registerConfiguration(container: DependencyContainer, configuration: Configuration) {
        container.register(.singleton) { configuration }
    }
    
    private func registerRepositories(container: DependencyContainer) {
        container.register { TasksRepository(configuration: $0) as TasksRepositoryProtocol }
        container.register { UsersRepository(configuration: $0) as UsersRepositoryProtocol }
    }
}
```

I’m registering configuration as a singleton. Now we can resolve our configuration for example by injection to the constructor:

```swift
class TasksRepository : TasksRepositoryProtocol {
        
    init(configuration: Configuration) {
        print("Database host: \(configuration.databaseHost)")
    }
}
```

Thus in my `TaskRepository` I have injected configuration object and I can use all properties from that object.

---

Now I have implemented mechanism which I can use for storing access information to the database. In next article I will use that information to prepare connection and read/write objects from/to real database (with some ORM framework).

---

All source codes you can find in my GitHub project (branch `configuration`): [mczachurski/TaskServerSwift](https://github.com/mczachurski/TaskServerSwift).