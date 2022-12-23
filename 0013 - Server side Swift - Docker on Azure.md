# Server side Swift - Docker on Azure

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0033.png)

> This is the article created at Mar 8, 2018 and moved from Medium.

My previous articles described how I realized MVC pattern in server side Swift HTTP application. Now we will create Docker file, push it to the Docker hub and create Web App for Containers on Azure.
<!--more-->

---

#### Create Dockerfile

First we have to install [Docker for your system](https://store.docker.com/search?type=edition&offering=community). When we have Docker ready we have to create simple configuration file: `Dockerfile`. This file is like a recipe — we have to get some base image, install new application _and add a pinch of this and that._

My `Dockerfile` file looks like on following snippet.

```
FROM swift:4.0.3
LABEL Description="TaskServer (swift) running on Docker" Vendor="Marcin Czachurski" Version="1.0"
RUN apt-get update \
    && apt-get install -y openssl libssl-dev uuid-dev sqlite3 libsqlite3-dev --no-install-recommends
ADD . /server
WORKDIR /server
RUN swift build --configuration release
EXPOSE 8181
ENTRYPOINT .build/release/TaskerServerApp
```

- `FROM` — sets our base image. [swiftdocker/swift](https://github.com/swiftdocker/docker-swift) is an Ubuntu 16.04 image with swift (4.0.3) installed.
- `LABEL` — the `LABEL` instruction adds metadata to an image. A `LABEL` is a key-value pair.
- `ADD` — adds the content of a local directory to a directory inside the image. We add the local directory `.`(the current directory) to the image’s `/server` directory.
- `WORKDIR` — sets directory from which `RUN` and `ENTRYPOINT` commands are run.
- `RUN` — this will execute a command. Here we build the Swift project in _release_ configuration.
- `EXPOSE` — informs Docker that the container listens on the specified network port at runtime. We expose port 8181 which our application runs on.
- `ENTRYPOINT` — is the executable that our container will run. We us `.build/release/TaskServerApp` which was built in the `RUN` step.

Here you can find detailed description of `Dockerfile` format and Docker commands: [Docker Documentation](https://docs.docker.com/engine/reference/builder/).

We have to put this file in root folder of our application (the same folder which contains `Package.swift` file for example).

#### Create `.dockerignore` file

Now we should create file named `.dockerignore`. That file will contain folders which we shouldn’t copy to the Docker container. Of course we can skip this step, but that file are not necessary from Docker perspective and simply creating our Docker image will be faster. My `.dockerignore` looks like on following snippet.

```
.git/
.build/
!.build/checkouts/
!.build/repositories/
!.build/workspace-state.json
```

#### Build image

We have everything in place to build our Docker image. For that purpose we have to run following command.

```sh
$ docker build --tag [username]/taskerserverswift .
```

Where `[username]` is your user name on [https://hub.docker.com/](https://hub.docker.com/). Of course if you would like to sent that image to the cloud. Otherwise you can skip that part of tag.

#### Run image

Now we are ready to run our image on local machine. We can execute following command.

```sh
$ docker run --name webserver -p 80:8181 [username]/taskerserverswift
```

If everything has been run correctly we can open in browser address: [http://localhost/](http://localhost/health) and we should got a response from our Swift server hosted as an image Docker on Ubuntu server. As you can notice, above command redirects external port `80` to internal port `8181` which we exposed in `Dockerfile` file.

#### Push container to Docker Hub

Later in the article I will create _Web app_ based on Docker image from Docker hub. However first we have to push our image to that hub (of course first you have to create account here: [https://hub.docker.com/](https://hub.docker.com/)). Sending our image to cloud is pretty easy. You have to execute command:

```sh
$ docker push [username]/taskerserver:v1.0.0
```

And now we can use that image for new application in Azure.

#### Deploy image to Azure

Deploying images to Azure is really simple. You have to click _Create a resource_ and choose: _Containers_ > _Web app for Containers_.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0034.png)

And then we have to enter some basic information like: _App name_, _Resource group_ and _App service plan_. After that we can configure place from Azure will get our Docker image. Here ee can choose _Docker hub_ and enter our image tag.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0035.png)

After that Azure should create new application. As you can see creating and deploying Docker image to the Azure cloud is very easy.

---

All source codes you can find in my GitHub project (branch `docker`): [mczachurski/TaskServerSwift](https://github.com/mczachurski/TaskServerSwift).