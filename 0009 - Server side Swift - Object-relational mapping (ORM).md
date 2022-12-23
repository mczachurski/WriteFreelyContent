# Server side Swift - Object-relational mapping (ORM)

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0021.jpeg)

> This is the article created at Feb 22, 2018 and moved from Medium.

All data which we are collecting in our application have to be stored in some place. Most common way is to use database server. Depending on application (problem which we have to solve) we can choose SQL or NoSQL database. Here I will focus on SQL (relational) databases. Most known SQL databases are: [Microsoft SQL Server](http://www.microsoft.com/sqlserver/), [Oracle](https://www.oracle.com/database/index.html), [MySQL](https://www.mysql.com), [MariaDB](https://mariadb.com), [PostgreSQL](https://www.postgresql.org), [Firebird](http://www.firebirdsql.org).
<!--more-->

In our application we can run common SQL queries to read/write to the SQL server. However in our case this is not optimal solution (and in most cases it’s not optimal). We will have objects which can be simply mapped to the database tables. When we will use SQL a lot of code will be very similar and we will copy and paste most of them. Instead of that we can use special libraries: object relational mapping frameworks, which automatically will map our objects to the database tables.

Features which modern ORM shoud have implemented:

- **Mapping between objects and database’s tables with CRUD support** — this is obvious, it’s most important feature of each ORM, thanks to this we can read/add/update/delete object data from/to database server.
- **Querying data** — ORM framework should provide solution to create queries using specific language syntax (we shouldn’t use plain SQL syntax in our source code). In C# we can use Linq for preparing queries, under the hood that queries are translated by Entity Framework to the correspondent SQL queries.
- **Relationships** — defining relationships between objects, thanks to this database can create foreign keys (or even separe tables when we have many to many relationships). ORM framework can use that information also for features like lazy loading or eager loading.
- **Session level cache** — this is feature which has big impact on performance. During one session (in many implementations session is created for one HTTP request) we can cache objects retrieved from database. Thanks to this ORM will reduce loopbacks to the database server.
- **Executing custom queries** — sometimes there is a need to execute custom SQL queries. For example when we would like to run batch queries like changing state for a significant number of rows. ORM should provide methods for that purposes.
- **Transaction support** — framework should provide API to use database transactions. This is very helpfull when we would like to save a few different objects during one HTTP request and database should be consistent.
- **Numerous databases supported** — ORM should support most important databases. Thanks to this our customers can have posibilities to choose database server which they know/like.
- **Migrations** — ORM should provide for us mechnism which we can use to prepare migrations between model versions.

In .NET world we have at least two very powerful ORM frameworks: [NHibernate](http://nhibernate.info) and [Entity Framework](https://docs.microsoft.com/en-us/ef/). Both fulfill all above requirements. Unfortunately in Swift world it is not so colorful.

I found frameworks:

- [StORM](https://github.com/SwiftORM) — StORM is a modular ORM for Swift, layered on top of [Perfect](https://github.com/PerfectlySoft/Perfect). It supports: [SQLite3](https://www.perfect.org/docs/StORM-SQLite.html), [PostgreSQL](https://www.perfect.org/docs/StORM-PostgreSQL.html), [MySQL](https://www.perfect.org/docs/StORM-MySQL.html), [Apache CouchDB](https://www.perfect.org/docs/StORM-CouchDB.html), [MongoDB](https://www.perfect.org/docs/StORM-MongoDB.html).
- [Fluent](https://docs.vapor.codes/2.0/fluent/getting-started/) — on project’s documentation page we can read: “Fluent provides an easy, simple, and safe API for working with your persisted data”. It supports: [Memory](https://docs.vapor.codes/2.0/fluent/package/), [SQLite3](https://docs.vapor.codes/2.0/fluent/package/), [MySQL](https://docs.vapor.codes/2.0/mysql/package/), [PostgreSQL](http://PostgreSQL) (not official), [MongoDB](https://github.com/vapor-community/mongo-provider) (not official). Its framework created under [Vapor](https://docs.vapor.codes/2.0/) project.
- [Perfect CRUD](https://github.com/PerfectlySoft/Perfect-CRUD) — this is pretty new project (not production ready yet) but very promising. It takes Swift 4 `Codable` types and maps them to SQL database tables. Currently it supports: [SQLite3](https://github.com/kjessup/Perfect-SQLite) and [PostgreSQL](https://github.com/kjessup/Perfect-PostgreSQL).

In StORM and Fluent I didn’t like how the ORM is realized. We have to inherit our model classes from special classes provided by ORMs. In StORM it’s for example `PostgresStORM` or `SQLiteStORM` etc. (depending on database provider). In Fluent we have to inherit from `Model`. Then access to the database is realized by our class. Saving data in both frameworks looks very similar.

StORM:

```swift
let obj         = User()
obj.firstname   = "Joe"
obj.lastname    = "Smith"
try obj.save {
    id in obj.id = id as! Int
}
```

Fluent:

```swift
let dog = Pet(name: "Spud", age: 5)
try dog.save()
print(dog.id) // the newly saved pet's id
```

Queries are realized a little bit different. Below I put examples how they looks in both frameworks.

StORM:

```swift
let obj = User()
try obj.find([("firstname", "Joe")])
print("Find Record:  \(obj.id), \(obj.firstname), \(obj.lastname)")
```

Fluent:

```swift
let dogs = try Pet.makeQuery().filter("age", .greaterThan, 2).all()
print(dogs) // all dogs older than 2
```

As you can see access to the database is realized by model classes. In StORM when I would like to retrieve collection of some entities first I have to instantiate object. Thus when I would like to retrieve `users` list first I have to create `User()` object. It’s for me really strange solution, which probably was created because of restrictions of previous versions of Swift.

Fluent seems to be much more powerful then StORM. It supports (simple) migrations, transactions, relations. All that features is missing in StORM.

Perfect CRUD is realized in a different way. We have separated logic between ORM features and our model. CRUD delivers `Database` class which provides abstraction level between our model and the database (we can compare it to the `DBContext` from Entity Framework). We can use that class to create tables into the database (additionally we have option to add/remove separate columns in the table). Also we can use `Database` class to get references to the `Table` class (which is a little bit similar to `DBSet` from Entity Framework). We can use that class when we would like to create queries for example.

Below is a simple example what we have to do when we would like to use Perfect CRUD.

```swift
struct PhoneNumber: Codable {
	let id: UUID
	let personId: UUID
	let code: Int
	let number: String
}
struct Person: Codable {
    let id: UUID
    let name: String
    let phoneNumbers: [PhoneNumber]?
}
let sqlite = try SQLiteDatabaseConfiguration(dbName)
let db = Database(configuration: sqlite)
try db.create(Person.self, policy: .reconcileTable)
let personTable = db.table(Person.self)
// Create.
let john = Person(id: 1, name: "John Doe")
let victoria = Person(id: 2, name: "Victoria Doe")
try personTable.insert([john, victoria])
// Queries.
let query = try personTable
    .order(by: \.name)
    .join(\.phoneNumbers, on: \.id, equals: \.personId)
    .order(descending: \.code)
    .where(\Person.name == "John" && \PhoneNumber.code == 12)
    .select()
```

More complex examples you can find on [Perfect CRUD GitHub page](https://github.com/PerfectlySoft/Perfect-CRUD).

In the further content of the article I will present how I included Perfect CRUD framework into my project. I can use that framework because my application is not created for production purposes (yet). I hope that framework will be production ready as soon as possible because his concepts and syntax looks best for me.

---

#### Dependencies

We have to add Perfect CRUD (I will use SQLite database) to the required packages and to the proper dependencies.

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
        .package(url: "https://github.com/mczachurski/Dobby", from: "0.7.1"),
        .package(url: "https://github.com/IBM-Swift/Configuration.git", from: "3.0.1"),
        .package(url: "https://github.com/kjessup/Perfect-SQLite.git", .branch("master"))
    ],
    targets: [
        // Targets are the basic building blocks of a package. A target can define a module or a test suite.
        // Targets can depend on other targets in this package, and on products in packages which this package depends on.
        .target(name: "TaskerServerApp", dependencies: ["TaskerServerLib", "PerfectHTTPServer", "Dip", "Configuration", "PerfectSQLite"]),
        .target(name: "TaskerServerLib", dependencies: ["PerfectHTTPServer", "Dip", "Configuration", "PerfectSQLite"]),
        .testTarget(name: "TaskerServerLibTests", dependencies: ["TaskerServerLib", "PerfectHTTPServer", "Dip", "Configuration", "Dobby"])
    ]
)
```

When we put everything in place we have to run following commands:

```sh
$ swift build  
$ swift package generate-xcodeproj
```

Thanks to this proper libraries are downloaded and new `xcodeproj` file is generated.

#### DatabaseContext

EntityFramework provides class `DbContext` which is main endpoint to getting in to the database. For each table we have to add corresponding`DbSet`. Also we can configure in `DbContext` database model (like table names, indexes, foreign keys etc.). I would like to introduce similar class in my application. And then I would like to inject this class to my repositories.

First I have to create protocol which I will add to my DI container and inject to my repositories.

```swift
public protocol DatabaseContextProtocol {
    func executeMigrations()
    func set<T: Codable>(_ type: T.Type) -> Table<T, Database<SQLiteDatabaseConfiguration>>
}
```

I have two simple functions here:

- `executeMigrations` is responsible for creating database with all tables (also if we add a net property to the class a new column will be added to the database)
- `set<T: Codable>` is generic function which can return for us object which we can use to access to the specific table.

My implementation of `DatabaseContextProtocol` looks like this:

```swift
public class DatabaseContext: DatabaseContextProtocol {

    private var sqlConnection: SqlConnectionProtocol
    private var database: Database<SQLiteDatabaseConfiguration>
    private let lock = NSLock()
    
    public func set<T: Codable>(_ type: T.Type) -> Table<T, Database<SQLiteDatabaseConfiguration>> {
        self.validateConnection()
        return database.table(type)
    }
    
    init(sqlConnection: SqlConnectionProtocol) {
        self.sqlConnection = sqlConnection
        let databaseConfiguration = sqlConnection.getDatabaseConfiguration() as! SQLiteDatabaseConfiguration
        self.database = Database(configuration: databaseConfiguration)
    }
    
    private func validateConnection() {
        
        if self.sqlConnection.isValidConnection() {
            return
        }
        
        lock.lock()
        if !self.sqlConnection.isValidConnection() {
            let databaseConfiguration = sqlConnection.getDatabaseConfiguration() as! SQLiteDatabaseConfiguration
            self.database = Database(configuration: databaseConfiguration)
        }
        lock.unlock()
    }
    
    public func executeMigrations() {
        self.validateConnection()
        
        try! self.database.create(Task.self, policy: .reconcileTable)
        try! self.database.create(User.self, policy: .reconcileTable)
    }
}
```

As you can see there is no magic here. Maybe you have to pay attention to `validateConnection()` function. I’m calling this function before any request to the database. If connection to the database is lost then we have to connect to the database again (we have to use locks if we would like to have that class thread safety).

I’m injecting to the `DatabaseContext` object which is responsible for holding real connection to the database. We should have only one instance of that class in the application (that’s why this object is registered in DI container as a singleton). `DatabaseContext` object can be created (resolved) multiple times.

```swift
public protocol SqlConnectionProtocol {
    func getDatabaseConfiguration() -> DatabaseConfigurationProtocol
    func isValidConnection() -> Bool
}

class SQLiteConnection : SqlConnectionProtocol {

    private let connectionString: String
    private var configuration: SQLiteDatabaseConfiguration?
    private let lock = NSLock()
    
    public func getDatabaseConfiguration() -> DatabaseConfigurationProtocol {

        if self.configuration != nil && isValidConnection() {
            return self.configuration!
        }
        
        lock.lock()
        if self.configuration == nil {
            self.configuration = try! SQLiteDatabaseConfiguration(connectionString)
        }
        lock.unlock()
        
        return self.configuration!
    }
    
    init(configuration: Configuration) {
        self.connectionString = configuration.connectionString
    }
    
    public func isValidConnection() -> Bool {
        // Here we should verify state of connection.
        return true
    }
}
```

That class basing on `Configuration` is establishing connection to the database. Also we have simple function which verify connection to the database (in this example connection to the SQLite file is always valid).

#### Dependency injection

Now we have to add our two new classes to the dependency injection container. I’m doing this in my `DependencyContainer` extension. Besides this we have to inject `DatanaseContext` to our repositories.

```swift
  private func registerDatabase(container: DependencyContainer) {
      container.register(.singleton) { SQLiteConnection(configuration: $0) as SqlConnectionProtocol }
      container.register { DatabaseContext(sqlConnection: $0) as DatabaseContextProtocol }
  }

  private func registerRepositories(container: DependencyContainer) {
      container.register { TasksRepository(databaseContext: $0) as TasksRepositoryProtocol }
      container.register { UsersRepository(databaseContext: $0) as UsersRepositoryProtocol }
  }
```

#### Migrations

Now in our `main.swift` class we can execute migrations. We can do this by executing following code.

```swift
// Run migrations.
let databaseContext = try! container.resolve() as DatabaseContextProtocol
databaseContext.executeMigrations()
```

If we are running application first time we should have in the console following messages:

```
[INFO] Starting HTTP server www.example.com on :::8181
[czw., 22 lut 2018 18:01:29 +0100] [QUERY] CREATE TABLE IF NOT EXISTS "Task" (
 id INT PRIMARY KEY,
 name TEXT NOT NULL,
 isFinished INT NOT NULL
)
[czw., 22 lut 2018 18:01:29 +0100] [QUERY] CREATE TABLE IF NOT EXISTS "User" (
 id INT PRIMARY KEY,
 name TEXT NOT NULL,
 email TEXT NOT NULL,
 isLocked INT NOT NULL
)
```

New tables are created. If we add a new column to our model and run again apllication on console we should see following messages.

```
[INFO] Starting HTTP server www.example.com on :::8181
[czw., 22 lut 2018 18:02:51 +0100] [QUERY] ALTER TABLE "Task" ADD COLUMN details TEXT
```

Thus we can see that Perfect CRUD executes `ALTER TABLE` statement. A little bit different situation looks when we are removing property from table.

```
[INFO] Starting HTTP server www.example.com on :::8181
[czw., 22 lut 2018 18:03:29 +0100] [QUERY] ALTER TABLE "Task" RENAME TO "temp_Task_temp"
[czw., 22 lut 2018 18:03:29 +0100] [QUERY] CREATE TABLE IF NOT EXISTS "Task" (
id INT PRIMARY KEY,
 name TEXT NOT NULL,
 isFinished INT NOT NULL
)
[czw., 22 lut 2018 18:03:29 +0100] [QUERY] INSERT INTO "Task" (id,name,isFinished)
SELECT id,name,isFinished
FROM "temp_Task_temp"
[czw., 22 lut 2018 18:03:29 +0100] [QUERY] DROP TABLE "temp_Task_temp"
```

As you can see our table is renamed, and then new table (with new schema) is created. Then all data from temp table is copied to the new table (and temp table is removed).

Thanks to this statements our model and tables in database are always consistent.

#### Repositories

The last thing is to replace old implementation of repositories (with hardcoded list) by impementation which are using `DatabaseContext`.

```swift
class TasksRepository : TasksRepositoryProtocol {
    
    private let databaseContext: DatabaseContextProtocol
    
    init(databaseContext: DatabaseContextProtocol) {
        self.databaseContext = databaseContext
    }
    
    func getTasks() throws -> [Task] {
        let tasks = try self.databaseContext.set(Task.self).select()
        return tasks.sorted { (task1, task2) -> Bool in
            return task1.name < task2.name
        }
    }
    
    func getTask(id: Int) throws -> Task? {
        let task = try self.databaseContext.set(Task.self).where(\Task.id == id).first()
        return task
    }
    
    func addTask(task: Task) throws {
        try self.databaseContext.set(Task.self).insert(task)
    }
    
    func updateTask(task: Task) throws {
        try self.databaseContext.set(Task.self).where(\Task.id == task.id).update(task)
        
    }
    
    func deleteTask(id: Int) throws {
        try self.databaseContext.set(Task.self).where(\Task.id == id).delete()
    }
}
```

---

Now we have all pieces of puzzle in place. During starting application we are executing migrations. To our repositories we are injecting class which is responsible for accessig database.

This was possible because Perfect CRUD provides really good API which we can easily use. I hope that framework will be production ready soon.

---

All source codes you can find in my GitHub project (branch `orm`): [mczachurski/TaskServerSwift](https://github.com/mczachurski/TaskServerSwift).