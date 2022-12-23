# Swift - JSON Performance

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0073.png)

> This is the article created at Apr 26, 2018 and moved from Medium.

Which Swift framework for handling JSON encoding/decoding is the fastest? Is there any faster then built in algorithms which operates on Codable protocol? And how we compare them to the .NET Core implementations?
<!--more-->

---

As you may know I’m trying to figure out how can I build, in most efficient way, server side application in Swift. I chose Perfect framework for my application and I have already built some example [project](https://github.com/mczachurski/TaskServerSwift). I wanted to know how I can compare performance between that solution and very similar written in .NET Core. I expected that application created in programming language which is compiling to native code will be faster than interpreted by .NET runtime. I described my experiment [here](https://medium.com/@mczachurski/swift-vs-net-core-benchmark-6153f1052b68). However I decided not to give up and try to find framework for JSON which will be fast enough and comparable to .NET application.

---

#### Frameworks

I’ve found a few frameworks on GitHub. I chose some of them (those which in my opinion were the most interesting one) to check performance.

- [HandyJSON](https://github.com/alibaba/HandyJSON)
- [Marshall](https://github.com/utahiosmac/Marshal)
- [ObjectMapper](https://github.com/Hearst-DD/ObjectMapper)
- [PMJSON](https://github.com/postmates/PMJSON)
- [SwiftProtobuf](https://github.com/apple/swift-protobuf.git)
- Codable — is built in Swift encoding & decoding based on Codable protocol
- NetCore — is .NET Core implementation of JSON decoding/encoding (Newtonsoft.JSON framework)

Later in the article I’ve posted the results of my tests.

---

#### Single object

Below there is a chart which visualizes benchmarks of decoding & encoding single object (which is serialize/deserialize 10,000 times).

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0074.png)

As you can see .NET Core implementation (Newtonsoft.JSON) is very fast. However Swift Protobuf is even faster. Wow. I didn't expected that. And this test confirmed that JSON encoding/decoding based on Codable protocol is really slow.

---

#### List of objects

Second chart shows benchmarks of encoding/decoding list of objects. One hundred objects are serialized/deserialized 10,000 times.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0075.png)

That test confirmed that Swift Protobuf is very fast. This time it’s a bit slower than .NET Core framework but insignificantly.

All benchmark results and source code of benchmarks you can find here: [https://github.com/mczachurski/SwiftBenchmarkJSON](https://github.com/mczachurski/SwiftBenchmarkJSON)

---

I will use Swift Protobuf in my application (and of course I will prepare benchmarks again). Especially that this framework provides additional feature. Based on HTTP header (`application/json`, `application/octet-stream`) we can use binary data instead of JSON string. Thanks to this clients which support only JSON will use that kind of data with communication with server side. However clients which support Protobuf binary can use that protocol (which is faster and produce smaller messages — which is important when we send them by slow Internet connection).

#### Single object

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0076.png)

#### List of objects

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0077.png)

---

After above benchmarks I started to believe again that I can use Swift in my server side project. However I will know for sure after another tests conducted on my real application. I will let you know about the results.