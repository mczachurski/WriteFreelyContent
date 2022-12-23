# Kubernetes on Raspberry Pi with .NET Core

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0004.jpeg)

> This is the article created at Dec 27, 2017 and moved from Medium.

Few weeks ago I read an article written by Scott Hanselman about [Kubernetes on Raspberry Pi](https://www.hanselman.com/blog/HowToBuildAKubernetesClusterWithARMRaspberryPiThenRunNETCoreOnOpenFaas.aspx). I thought that it would be great to build similar infrastructure on own. Thanks to Unit4, where I’m working, it was possible. I ordered all required parts and my company covered all costs.
<!--more-->

My order:

- 6 x Raspberry Pi 3 Model B
- 6 x SAMSUNG EVO+ microSD 32GB (memory card)
- AUKEY CB-D10 (6 microUSB cables)
- AUKEY PA-T11 6xUSB 60W (USB charger)
- case (7 plastic layers for open case)

Besides that parts we have to have:

- monitor with HDMI
- USB keyboard

Next day I had all parts (except case) on my desk and I could start to build the cluster with Raspberries.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0005.jpeg)

Below you can find all steps which I had to do to build Kubernetes cluster infrastructure:

- [Prepare SD cards](#prepare-sd-cards)
- [Prepare Raspberries](#prepare-raspberries)
- [Configure Kubernetes master node](#configure-kubernetes-master-node)
- [Configure Kubernetes nodes](#configure-kubernetes-nodes)
- [Configure local computer](#configure-local-computer)
- [Kubernetes dashboard](#kubernetes-dashboard)
- [Deploying application](#deploying-application)

Our cluster will look like on following image.

![](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0006.png)

We have one master node `pi01` and five connected to Kubernetes cluster.

**So let’s do it!**

---

#### Prepare SD cards

In first step we have to prepare microSD cards. We can use the Raspbian Stretch Lite image downloaded from that site: [https://www.raspberrypi.org/downloads/raspbian/](https://www.raspberrypi.org/downloads/raspbian/).

Next we have to copy that image to the microSD cards. There are at least two great softwares for macOS which we can chose:

- Etcher — [https://etcher.io](https://etcher.io) (available also for Windows and Linux)
- ApplePi Baker — [https://www.tweaking4all.com/hardware/raspberry-pi/macosx-apple-pi-baker/](https://www.tweaking4all.com/hardware/raspberry-pi/macosx-apple-pi-baker/)

I prepared cards with Etcher which looks more like native macOS application for me. Then we have to boot all our mini computers.

---

#### **Prepare Raspberries**

Now we can power on our Raspberries, connect keyboard and monitor, and set up required settings. Default user name is `pi` and password is `raspberry`.

**Keyboard layout** _(optional step)_

First I had to change my keyboard layout. On default Raspbian runs with UK layout however I have US keyboard. I had to run following command:

```sh
$ sudo dpkg-reconfigure keyboard-configuration
```

Then we need to chose “Generic 105 keys” keyboard and language English (US).

**WiFi connection** (optional step if you can connect Raspberries to router by ethernet cables)

All my nodes are connected to network by WiFi. It’s not best solution, however I didn’t have enought ethernet cables. I had to do a few steps. First I had to configure default network interface. Edit file `/etc/network/interfaces` and add following lines:

```
auto wlan0  
iface wlan0 inet manual  
wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
```

Next we have to configure access to WiFi network which is in `/etc/wpa_supplicant/wpa_supplicant.conf` file. That file should looks like this:

```
country=GB  
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev  
update_config=1  
network={  
    ssid="your ssid"  
    psk="your wifi password"  
    key_mgmt=WPA-PSK  
}
```

Of course we have to enter proper network SSID and password. If you don’t want to enter password in plain text you can run command:

```sh
$ wpa_passphrase "you ssid" "your wifi password"
```

This command should return something like this:

```
network={
    ssid="you ssid"
    #psk="your wifi password"
    psk=238c7d2c09814cbf8c98a30773e4e2d221a723c38a1d0f0ba1e2e
}
```

Now instead plain text you can enter “encrypted” password (notice that there is no _“_ sign).

**Static IP address**

In next step we have to configure static IP addresses. It is best choice for our cluster. Edit file `/etc/dhcpcd.conf` and enter the following lines:

```
interface wlan0  
static ip_address=192.168.1.101/24  
static routers=192.168.1.10  
static domain_name_servers=62.179.1.63
```

You have to enter IP addresses which are correct in your network. `ip_address` is IP address of your Raspberry computer (each Raspberry should have different one). `routers` is address to your gateway/router. `domain_name_servers` is address of your DNS server (I choose DNS from my ISP company).

**Hostname**

Each of our mini computers have to have different host name. This is important for Kubernetes because we will see that name in a few places in Kubernetes dashboard. We have to run command:

```sh
$ sudo rasbpi-config
```

Choose “network” option and change host name. Remember that each node have to have different name.

**Docker**

Next we need to install [Docker](https://www.docker.com). This is container engine which Kubernetes uses to run inside the`Pods`. Instead of Docker you can also use [rkt](https://coreos.com/rkt/) (“_rocket_”) which is alternative container engine for Kubernetes and probably sometime it will be main container engine in Kubernetes world.

Commands to install Docker:

```sh
$ curl -sSL get.docker.com | sh  
$ sudo usermod pi -aG docker
```

**Disable Swap**

Now we have to disable swap on Linux. This is because of errors which Kubernetes throws. Execute following commands:

```sh
$ sudo dphys-swapfile swapoff  
$ sudo dphys-swapfile uninstall  
$ sudo update-rc.d dphys-swapfile remove
```

**Configure CGroups**

We have to change also some settings connected with cgroups (which provide a mechanism for easily managing and monitoring system resources, by partitioning things like cpu time, system memory, disk and network bandwidth, into groups, then assigning tasks to those groups). Edit file `/boot/cmdline.txt` and add at the end following text:

```
cgroup_enable=cpuset cgroup_memory=1
```

Without this settings we will have troubles with initialization of Kubernetes master node.

**Enable SSH**

It would be really paintfull uses monitor and keybord connected to each Raspberry. We have other, better option. We can enable SSH shell and connect from our local computer (in my case laptop with macOS) to the specific computer in the cluster. To enable SSH we have to run command:

```sh
$ sudo raspi-config
```

and we have to navigate to “Interfacing options” and then to “SSH”. After that we can connect to Raspberries from our local terminal by that command:

```sh
$ ssh pi@192.168.1.101
```

Of course we have to enter proper IP address.

**Install Kubernetes**

Now we have to install Kubernetes applications.

```sh
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -  
$ echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list  
$ sudo apt-get update -q  
$ sudo apt-get install -qy kubeadm
```

Now on each node we should have required applications to run Kubernetes cluster and we can start real fun!

---

#### Configure Kubernetes master node

Now we can chose one computer which will have special tasks. This computer will be our master node. On that computer we have to run command:

```sh
$ sudo kubeadm init --apiserver-advertise-address=192.168.1.101
```

IP address is address of our master node (so computer where we execute that command). On console we should have result similar like this (you should save that response, we will use some of commands on other nodes):

```
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

mkdir -p $HOME/.kube  
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config  
sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.

Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at: [https://kubernetes.io/docs/concepts/cluster-administration/addons/](https://kubernetes.io/docs/concepts/cluster-administration/addons/)

You can now join any number of machines by running the following on each node as root:

sudo kubeadm join --token c5334d.24da95e5e961a5af 192.168.1.101:6443 --discovery-token-ca-cert-hash sha256:acbae0a1b8a12496cd1de3fb750e36c53179d8a014ed51717fc1b89888c28a66
```

From that response we know that we have to run following commands on the current (master) node:

```sh
$ mkdir -p $HOME/.kube  
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config  
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Before we will execute next commands (join) we have to do one more thing on master node: installing network driver. This is mandatory step and without doing this nodes cannot go to “Ready” state (communication between nodes will not work). Run following command on master node:

```sh
$ kubectl apply -f https://git.io/weave-kube-1.6
```

Now we can verify number of nodes in our Kubernetes cluster:

```sh
$ kubectl get nodes
```

We should get something like this on console:

```
NAME      STATUS    ROLES     AGE       VERSION  
pi01      Ready     master    19h       v1.9.0
```


#### Configure Kubernetes nodes

Now we can connect rest of our mini computers to the cluster (master node). Execute following command on each of our Raspberries (besides master node):

```sh
sudo kubeadm join --token c5334d.24da95e5e961a5af 192.168.1.101:6443 --discovery-token-ca-cert-hash sha256:acbae0a1b8a12496cd1de3fb750e36c53179d8a014ed51717fc1b89888c28a66
```

Now we can verify number of nodes in our Kubernetes cluster:

```sh
$ kubectl get nodes
```

We should get something like this:

```
NAME      STATUS    ROLES     AGE       VERSION  
pi01      Ready     master    19h       v1.9.0  
pi02      Ready     <none>    19h       v1.9.0  
pi03      Ready     <none>    19h       v1.9.0  
pi04      Ready     <none>    16h       v1.9.0  
pi05      Ready     <none>    16h       v1.9.0  
pi06      Ready     <none>    16h       v1.9.0
```

From the above response we see that I have six nodes (including master node) connected in cluster. All of them are in `Ready` state.

---

#### Configure local computer

For now all commands we executed on nodes (most of them on master node). However we can control cluster from external computer, for example from our laptop. For this purpose we have to install on local computer `kubectl` application. On macOS we can execute following command:

```sh
$ brew install kubectl
```

Now we can verify installation:

```sh
$ kubectl version
```

We should get on console current version of the application. Now we have to copy from our master node Kubernetes configuration file. We can do this by execute following command:

```sh
$ scp pi@192.168.1.101:/home/pi/.kube/config .
```

Of course you have to enter proper IP address (address of your master node). Now we have to put that file in default place (place where `kubectl` search on default). Run following commands:

```sh
$ mkdir .kube  
$ mv config ./kube
```

Thus downloaded `config` file should be place in folder `/home/[user]/.kube/`. Now we cane run `kubectl` application:

```sh
$ kubectl get nodes
```

We should get the same response which we have previously, when we run that command on master node.

---

#### Kubernetes dashboard

Now we can install special `Pod` which contains web dashboard for Kubernetes cluster. It will be our first application installed in the cluster. That application is really helpful specially at the begging of our Kubernetes journey. Here we will use dashboard without best access control set up. I will install dashboard and add one user with admin privileges.

**Install dashboard**

Dashboard is simply the `Pod` which we have to install on Kubernetes cluster. Frist we have to download `yaml` file with dashboard metadata:

```sh
$ curl https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/alternative/kubernetes-dashboard-arm.yaml > dashboard.yaml
```

Now we can send that data to the Kubernates.

```sh
$ kubectl create -f dashboard.yaml
```

**Add user with admin privileges**

Now we have to add new user to the Kubernetes. We have to save following text to the file `create-admin-user.yaml`:

```yaml
apiVersion: v1  
kind: ServiceAccount  
metadata:  
  name: admin-user  
  namespace: kube-system
```

and execute following command:

```sh
$ kubectl create -f create-admin-user.yaml
```

Now we can add roles to the new user. Thus we have to save following text to the file `add-role.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1  
kind: ClusterRoleBinding  
metadata:  
  name: admin-user  
roleRef:  
  apiGroup: rbac.authorization.k8s.io  
  kind: ClusterRole  
  name: cluster-admin  
subjects:  
- kind: ServiceAccount  
  name: admin-user  
  namespace: kube-system
```

Now we can execute following command:

```sh
$ kubectl create -f add-role.yaml
```

**Verify dashboard**

We can have access to the dashboard from local computer. However first we have to connect our computer to the cluster (via SSH tunnel). But first we have to check IP address of dashboard `Pods`. For this purpose we have to execute command:

```sh
$ kubectl get svc --namespace kube-system
```

We should have response similar like this:

```
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE  
kube-dns               ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP   20h  
kubernetes-dashboard   ClusterIP   10.108.75.231   <none>        80/TCP          17h
```

We have IP `10.108.75.231` and Port `80` for dashboard. Now we have everything to create SSH tunnel:

```sh
$ sudo ssh -L 8080:10.108.75.231:80 pi@192.168.1.101
```

Now we can open in the browser address: [http://localhost:8080/.](http://localhost:8080/.) We should see sign-in page. We can use token authorization here, but first we have to execute following command:

```sh
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

Above command returns for us token which we can copy and use in dashboard sign in page.

```
Name:         admin-user-token-v454k  
Namespace:    kube-system  
Labels:       <none>  
Annotations:  kubernetes.io/service-account.name=admin-user

kubernetes.io/service-account.uid=f64be1d8-e807-11e7-aaa1-b827eb632aa7

Type:  kubernetes.io/service-account-token

Data

====  
ca.crt:     1025 bytes  
namespace:  11 bytes  
token:      eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXY0NTRrIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJmNjRiZTFkOC1lODA3LTExZTctYWFhMS1iODI3ZWI2MzJhYTciLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.n5Jtq0FAy9mi51OGEUIn6WwzyL24Cqy1ewSbeFJU19RG1_bTaKrM9TPEcx6GypANNT2uMPSriIh7rpH-yu0lbjxvTSkEH6RW_Y7jLYbJC5EfI4FgiMteyl_uslnlkS561ZiR8H-xpZybM2CYyuSJYHPFnDmT14lpr_iyGXuDkikfyEtiKbXg9DioVKdZF9xloCRBKDBllPFdtUs4digaMgLMK4ABxEGiSFgBF9mdnsk-qRCla-5ieu2tfU1L4_YDOpLBnSaP0FZcxWuo8hrb3mQ1ukwRRGvxLdXOIXDx2KYQaVDyWGhJJOy48ZCxfkQXbfWMm6WrX-RidDwpolfodg
```

After putting this token on sign in page we should have access to all dashboard pages.

Main Kubernetes dashboard page

![Main Kubernetes dashboard page](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0007.png)

List of nodes in Kubernetes dashboard

![List of nodes in Kubernetes dashboard](https://raw.githubusercontent.com/mczachurski/WriteFreelyContent/main/images/0008.png)

---

#### Deploying application

Now our Kubernetes cluster is ready to use. We can install `Pods` and see how master node manages the cluster.

**Create .NET Core API application**

First you have to have .NET Core SDK installed on your local computer. Visit Microsoft official [page](https://www.microsoft.com/net/learn/get-started/macos) and install SDK. Then you can execute following commands to create simple web API project:

```sh
$ mkdir netcoreapi  
$ cd netcoreapi  
$ dotnet new webapi
```

By executing following command web server (Kestrel) is started:

```sh
$ dotnet run
```

Now your application should run. You can verify this by visiting page: [http://localhost:5000/api/values](http://localhost:5000/api/values). You should have simple `JSON` array as an response.

If everything runs correctly we can create Docker container now.

**Docker container**

`Pods` is a simple file which contains information about application which we want to deploy. Except information like name, address, port etc. `Pod` file contains URL to the docker file. That’s why now we have to prepare information about our application for Docker. Thus you have to have Docker client installed on your computer. You can install Docker from [this](https://www.docker.com/community-edition) site. Also you need to have your own `Docker ID`. Create your account on this page: [https://cloud.docker.com](https://cloud.docker.com).

Now we have to create `Dockerfile` (in folder where we created our .NET Core application):

```
FROM microsoft/aspnetcore-build:2.0 AS build-env
WORKDIR /app

# Copy csproj and restore as distinct layers
COPY *.csproj ./
RUN dotnet restore

# Copy everything else and build
COPY . ./
RUN dotnet publish -c Release -o out -r linux-arm

# Build runtime image
FROM microsoft/dotnet:2.0.0-runtime-stretch-arm32v7

WORKDIR /app
COPY --from=build-env /app/out .
ENV ASPNETCORE_URLS http://+:80

EXPOSE 80
ENTRYPOINT ["dotnet", "netcoreapi.dll"]
```

Now we can build Docker image.

```sh
$ docker build -t netcoreapi .
```

Then we have to tag our new container.

```sh
$ docker tag netcoreapi mczachurski/netcoreapi:v001
```

Of course you should use your `Docker ID` and your own tag name. Now we have everything ready and we can send image to the cloud.

```sh
$ docker push mczachurski/netcoreapi:v002
```

**Deploy application to Kubernetes**

We’re almost there. Now we can deploy our application to the Kubernetes cluster. First we have to create file with deployment metadata:

```yaml
apiVersion: apps/v1beta2kind: Deployment  
metadata:  
  name: netcoreapi-deployment  
spec:  
  selector:  
    matchLabels:  
      app: netcoreapi  
  replicas: 2  
  template:  
    metadata:  
      labels:  
        app: netcoreapi  
    spec:  
      containers:  
      - name: netcoreapi  
        image: mczachurski/netcoreapi:v001  
        ports:  
        - containerPort: 80
```

Now we can deploy that file to the Kubernetes cluster.

```sh
$ kubectl create -f netcoreapi-deployment.yaml
```

After that command we can verify status of our deployment:

```sh
$ kubectl get pods  
$ kubectl describe deployment netcoreapi-deployment
```

Now our application should be deployed and run. You should notice one important setting in our deployment metadata: `replicas`. This is information about number of instances of our application which should run in the cluster.

Unfortunatelly each instance has own internal IP address. It would be really painful uses that IP addresses. Especially that they can be changed by Kubenetes for example when one of node was shuted down (Kubernetes then is restoring application on other node with new IP address). If we want to have one address with load balancer we have to deploy new object: Service.

**Expose application by Service**

First we have to prepare file with service description.

```yaml
kind: Service  
apiVersion: v1  
metadata:  
  name: netcoreapi  
spec:  
  selector:  
    app: netcoreapi  
  ports:  
  - protocol: TCP  
    port: 8002  
    targetPort: 80  
  externalIPs:  
  - 192.168.1.101
```

Next we have to send that file to the Kubernetes.

```sh
$ kubectl create -f netcoreapi-service.yaml
```

Now we can open address: [http://192.168.101:8002/](http://192.168.101:80/) on our local computer and we should have response from our service installed inside the cluster. Awesome!

---

Now you can create Docker files for all your microservies. Deploy them to your Kubernetes cluster and experiment. Of course if you don’t want to buy Raspberries you can use normal PC (with Linux :-)) or you can use Linux VM on Azure or other cloud provider. Most of steps are the same (even sipmpler becouse you don’t have to be warry about network stuff).

This is only introduction to Kubernetes which is really powerfull tool. There is really good introduction course on Pluralsight about Kubernetes and a lot of good stuff on Youtube.

---

Links:

- [Getting Started with Kubernetes](https://www.pluralsight.com/courses/getting-started-kubernetes)
- [Kubernetes](https://kubernetes.io)
- [Raspbian](http://www.raspbian.org)
- [Docker](https://www.docker.com)