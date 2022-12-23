# Server side Swift - Continuous Integration

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0036.png)

> This is the article created at Mar 14, 2018 and moved from Medium.

What is continuous integration? On [Wikipedia](https://en.wikipedia.org/wiki/Continuous_integration) we can read:

> In [software engineering](https://en.wikipedia.org/wiki/Software_engineering "Software engineering"), **continuous integration** (**CI**) is the practice of merging all developer working copies to a shared [mainline](https://en.wikipedia.org/wiki/Trunk_%28software%29 "Trunk (software)") several times a day.[\[1\]](https://en.wikipedia.org/wiki/Continuous_integration#cite_note-:0-1) [Grady Booch](https://en.wikipedia.org/wiki/Grady_Booch "Grady Booch") first named and proposed CI in [his 1991 method](https://en.wikipedia.org/wiki/Booch_method "Booch method"),[\[2\]](https://en.wikipedia.org/wiki/Continuous_integration#cite_note-2) although he did not advocate integrating several times a day. [Extreme programming](https://en.wikipedia.org/wiki/Extreme_programming "Extreme programming") (XP) adopted the concept of CI and did advocate integrating more than once per day — perhaps as many as tens of times per day.[\[3\]](https://en.wikipedia.org/wiki/Continuous_integration#cite_note-3)

Thus basically CI from our source code can build target application (executable) frequently (in best case after each commit). This can be done by several ways like simple scripts or by software created specially for that purpose. We should be notified about status of build process (especially when build fails) — and this is one of most important purposes of CI.
<!--more-->

I would like to connect my Swift project (which I’ve described in my previous articles) to some CI system. I don’t want to create new CI service (on premise) but I would like to use some existing online services. My continuous integration system should contains following features:

- **verify build** — system has to build source code and verify status of the compilation
- **run unit tests** — system has to execute all unit tests and notify about failing tests
- **verify code coverage** — system has to calculate code coeverage
- **check code complexity** — system has to calculate code complexity of source code

Unfortunately there is no one online system that fulfill all that requirements. However we can choose a set of services that together will support all required features. First we have to choose system which will build our code and run unit tests.

List of free (for open source projects) CI online services:

- [Travis CI](https://travis-ci.org/) — free for open source projects, supports macOS and Linux
- [CircleCI](https://circleci.com/) — the first container is free, macOS starts from $39/month
- [AppVeyor](http://www.appveyor.com) — free for open source projects, mostly for Windows developers (AppVeyor for Linux is currently in private beta)

It’s much esier when we are working on some most common programming language like Java/C# etc. For my C# projects I used AppVeyor service, however for this project I decided to use Travis CI which supports macOS for open-source projects for free.

For code coverage we can use service: [Codecov](https://codecov.io/) and for code complexity: [Codebeat](https://codebeat.co/) service.

I will describe below how I connected my source code with all that services.

---

### Build

First we have to create account on [http://travis-ci.org/](http://travis-ci.org/) (probably best choice is to connect Travis CI with our GitHub account) and do the following things:

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0037.png)

Image from https://travis-ci.org site.

First thing is that we have to go to the our profile page on Travis CI and enable repositories which we have to connect. After that we can start creating configuration for our repository.

#### Configuration

We have to create `.travis.yml` file in root of our project with configuration for Travis CI service. Configuration for my Swift application is really simple and looks like on following snippet.

```yaml
language: swift
os: osx
osx_image: xcode9.2
before_script:
- gem install xcpretty
script:
- swift package generate-xcodeproj
- set -o pipefail && xcodebuild -scheme TaskerServer-Package clean build | xcpretty
```

The most tricky part here is to choose correct scheme for `xcodebuild`. If you are not sure what schemes you have there is a command that give you list of available schemas: `xcodebuild -list`.

Also I added here `xcpretty` which improve log from build process (we will have more readable output). More information about `xcpretty` you can find [here](https://github.com/supermarin/xcpretty).

Now we have to go to our project site on Travis CI to verify our build status. For me that site looks like [here](https://travis-ci.org/mczachurski/TaskServerSwift). It is nice and clear.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0038.png)

That was **easy, peasy!** We have Swift application with CI connected and we wrote only **_ONE_** simple file.

#### Badge

It’s very common to put build status on project’s GitHub page. Thanks to this everybody interested in project can easily find links to CI services and quickly verify build status. Below is example how this badge looks like on my project’s page:

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0039.png)

With Travis CI this is very easy. [Here](https://docs.travis-ci.com/user/status-images/) you can find information how to add badge on your site.

---

### Tests

Now we can start configuring unit tests. We will also run unit tests on Travis CI. We have to modify a little bit `.travis.yml` file. Below is my configuration file after modification.

```yaml
language: swift
os: osx
osx_image: xcode9.2
before_script:
- gem install xcpretty
script:
- swift package generate-xcodeproj
- set -o pipefail && xcodebuild -scheme TaskerServer-Package clean build test | xcpretty
```

I’ve added only one word: `test` to `xcodebuild` command. Now on our project’s page in Travis CI we have result from build process and unit tests execution. **Nice!**

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0040.png)

---

### Code coverage

For code coverage we will use separate service: [https://codecov.io/](https://codecov.io/). Of course at the beginning we have to create account on that service. Again the fastest way is to connect our GitHub account with Codecov.

We can send report from Trevis CI to the codecov after build. For that purpose we have to again modify our `travis.yml` file.

```yaml
language: swift
os: osx
osx_image: xcode9.2
before_script:
- gem install xcpretty
script:
- swift package generate-xcodeproj
- set -o pipefail && xcodebuild -scheme TaskerServer-Package -enableCodeCoverage YES clean build test | xcpretty
after_success:
- bash <(curl -s https://codecov.io/bash) -J 'TaskerServerLib'
```

First we have to inform `xcodebuild` that we would like to have code coverage enabled. Second thing is new section `after_success`. Here we are using script prepared by Codecov which search for code coverage report files and sent them to their system where this files are parsed.

Report for my project looks like [here](https://codecov.io/gh/mczachurski/TaskServerSwift/tree/master/Sources/TaskerServerLib).

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0041.png)

It is nice report, where we can go deeper to single file. At file level we can even check if single file was covered by unit test.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0042.png)

**Awesome!** That was really easy and I have exactly what I expected.

#### Badge

Of course Codecov also provide for us badge icon which we can include on our site.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0043.png)

Information about badge snippet we can find on project’s settings page on Codecov service.

---

### Code complexity

Last service is Codebeat: [https://codebeat.co/](https://codebeat.co/). This service can read our source code and it trying to figure out places in our code which we can improve. Here also we have to create account (we can also connect our GitHub account).

After that we have to connect our GitHub repository to the Codebeat. It’s easy step which we have to do on Codebeat web page.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0044.png)

Service need some time to download source code and process it. For my application I have results like [here](https://codebeat.co/projects/github-com-mczachurski-taskserverswift-master).

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0045.png)

Here we can also go deeper to single function where we can verify exactly where we have place for improvement.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0046.png)

**Great!** Report looks really clear and nice. It’s exactly what I would like to have.

#### Badge

Also this service provides for us badge icon which we can put on our site.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0047.png)

Snippet for that you can find on project’s settings page on Codebeat.

---

Now I have all puzzles in place and my CI service for open source server side Swift project is ready. After each build I have information about build process, unit tests, code coverage and code complexity. If I did some mistake I have feedback about this really quickly and whole process is automatic.

---

If you are curious how I built my example server side Swift project please read my previous articles. All source code you can find in my GitHub project (branch `continuous-integration`): [mczachurski/TaskServerSwift](https://github.com/mczachurski/TaskServerSwift).