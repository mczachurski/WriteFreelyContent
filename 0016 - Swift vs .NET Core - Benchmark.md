# Swift vs .NET Core - Benchmark

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0050.png)

> This is the article created at Apr 16, 2018 and moved from Medium.

Before you choose language/platform for your server side application you should know if it fulfills your requirements. For that kind of application performance it’s especially important. I spent a few last weeks for building server side Swift application (example project which I described in my previous articles). But is it fast enough? And how fast is Perfect/Swift compared to more mature platforms like .NET? Can we choose Swift as a solution for server application which requires extraordinary performance?
<!--more-->

There are benchmarks which compares server side Swift frameworks and Node.js. For example below great article: [Updated Benchmarks for the Top Server-Side Swift Frameworks vs. Node.js](https://medium.com/@rymcol/updated-benchmarks-for-the-top-server-side-swift-frameworks-vs-node-js-9da4a0491eca).

But I didn’t found any comparison to the .NET (or .NET Core) which is really mature platform. I decided to build very similar application to [TaskerServerSwift](https://github.com/mczachurski/TaskServerSwift) but in .NET Core. [Here](https://github.com/mczachurski/TaskServerCore) you can find source code of that application.

Both applications have DI (dependency injection), ORM with SQLite provider (Perfect CRUD/Entity Framework). ASP.NET Core application also uses Automapper for mapping objects with DTOs. So I would like compare “real” business applications and not only some language specific stuff (like encrypting, algorithms etc.).

### Infrastructure

Both applications (Swift and .NET) I’ve uploaded as a Docker images to the Azure as a **_Web App for Containers_**. I’ve also created additional **_Virtual Machine_** in Azure (Ubuntu Server).

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0051.png)

I executed tests from my **MacBook** from my local network (I have 250 Mb/s connection) and from **VM** on Azure (dockers and VM are on the same Azure location: West Europe). For performing tests I uses [wrk](https://github.com/wg/wrk) (HTTP benchmarking tool). Command which I run:

```sh
$ wrk -d 10m -t 4 -c 20 https://tasker[swift/core].azurewebsites.net/tasks -H "Authorization: Bearer [TOKEN]"
```

Thus each test lasted 10 minutes and uses 4 thread and 20 connections.

#### Benchmark #1: Simple JSON

Both applications has endpoint: `/health`, which returns simple JSON:

```json
{
    "message": "I'm fine and running!"
}
```

I executed `wrk` tool few times and below there is a chart with results.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0052.png)

So far it seems very promising. Swift is faster then .NET Core. It’s what I expected from native application. On Azure **VM** I have **171.82** requests/seconds for Swift (**411057** requests in 10.00m, **124.66MB** read) and **112.04** requests/seconds for .NET Core (**267674** requests in 10.00m, **85.79MB** read). Pretty good.

#### Benchmark #2: List of tasks

Next test is a little bit more complicated. I added to both applications 100 `Task` objects and I’m using endpoint: `GET /tasks` for retrieving all that objects as a JSON response. My response should looks like on below snippet.

```json
[
    {
        "createDate": "2018-04-15T11:11:00:396",
        "name": "Task 001",
        "id": "8E8DA792-9746-4ABD-87DC-125FB65CD776",
        "isFinished": false
    },
    ...
]
```

Chart with results.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0053.png)

**WHAAAT!** I don’t understand. Impossible. On Azure **VM** I have **6.29** requests/seconds for Swift (**8951** requests in 10.00m, **96.30MB** read) and **26.39** requests/seconds for .NET Core (**62260** requests in 10.00m, **719.11MB** read). Why Swift application is so slow?

#### Benchmark #3: List of tasks (after fix issues)

My first thought was that probably this is because of access to the SQLite database. I removed access to the database and moved objects to the controller file (I always returned the same 100 `Task` objects) and I perform tests again.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0054.png)

It’s better but still disappointing. On Azure **VM** I have **14.03** requests/seconds for Swift (**31840** requests in 10.00m, **350.52MB** read). Still twice times slower then application in .NET which downloads objects from the database.

**Why?** In my application I have three section:

- create list of objects (download from SQLite or created statically in controller)
- serialize objects to JSON
- return JSON by Perfect framework

I verified that creating objects is super fast. Also from first benchmark we know that Perfect can return a lot of requests. Thus it seems that JSON serialization is our suspect.

#### Benchmark #4: Swift performance

I created new simple application. This application has only four endpoints which basically returns JSON. Source code of that application you can find on [GitHub](https://github.com/mczachurski/SwiftPerformance). Endpoints:

- Static JSON (`/endp1`) — this endpoint has hardcoded JSON string and it simply returns it (without any object to JSON serialization).
- JSONEncoder (`/endp2`) — this endpoint creates dynamically (in loop) one hundred `Task` objects and serialize them to JSON by `JSONEncoder` introduced in Swift 4.
- JSONEncoder static list (`/endp3`) — this endpoint is similar to the previous one, however list of objects is created once during starting the application.
- JSONEncodedString (`/endp4`) — this endpoint creates dynamically (in loop) one hundred `Task` objects and serialize them to JSON by `JSONEncodedString` extension provided by Perfect Server library.
- JSONEncodedString static list (`/endp5`) — this endpoint is similar to the previous one, however list of objects is created once during starting the application.

I pushed this application to the Docker hub and deployed it as a new _WebApp for Container_ in Azure. I performed tests from **VM** on Azure.

Below there is a performance chart.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0055.png)

As you can see Perfect can return **356555** requests in 10 minutes (**4.00GB** read). It’s almost **149.09** requests/second (and our server have only one core). However when we have to serialize object to JSON performance decrease dramatically. Now server can serve only **6–10** requests per second (depending on serialization type). **That speed is not acceptable.**

#### Benchmark #5: JSON Serialization

Now I know that serialization object to the JSON is not optimal. But how it looks compared to .NET Core. I created two simple console applications.

**Application in C#**

```cs
static void Main(string[] args)
{
    var list = GetTasks(); // returns 100 tasks
    for(int i = 1; i <= 100000; ++i) {
        var json = JsonConvert.SerializeObject(list);
    }
}
```

**Application in Swift**

```swift
var list = getList() // returns 100 tasks
let encoder = JSONEncoder()
do {
    for _ in 1...100000 {
        _ = try encoder.encode(list)
    }
} catch {
    print("Error during serialize object to JSON")
}
```

I’ve performed tests on my MacBook. Below there is benchmark result chart.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0056.png)

From above test we can see that 100 thousands of objects (each objects is a list of 100 objects) is serialized to JSON by .NET Core in **18.7** seconds. Swift is almost eight times slower. Serialization the same amount of objects takes **137.6** seconds. It’s really strange and disappointing.

---

For now I didn’t found any good solution (other faster framework) for improving serialization to JSON in Swift.

---

If my boss ask me if we can use Swift as a server side platform I have to answer: **NO**. Unfortunately, because Swift as a language is great for server application. Is very strict and much more safe then C#. Language and compiler make sure that the developer makes as few mistakes as possible.