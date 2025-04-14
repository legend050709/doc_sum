```table-of-contents
```

# Docker network的网络特性
Docker在1.9版本中引入了一整套的自定义网络命令和跨主机网络支持。这是libnetwork项目从Docker的主仓库抽离之后的一次重大变化。
libnetwork项目从lincontainer和Docker代码的分离早在Docker 1.7版本就已经完成了（从Docker 1.6版本的网络代码中抽离）。在此之后，容器的网络接口就成为了一个个可替换的插件模块。由于这次变化进行的十分平顺，作为Docker的使用者几乎不会感觉到其中的差异，然而这个改变为接下来的一系列扩展埋下了很好的伏笔。

## libnetwork
### CNM
libnetwork所做的最核心事情是定义了一组标准的容器网络模型（**Container Network Model，简称 CNM），只要符合这个模型的网络接口就能被用于容器之间通信，而通信的过程和细节可以完全由网络接口来实现。**'

Docker的容器网络模型最初是由思科公司员工Erik提出的设想。在这个网络模型中定义了三个的术语：Sandbox、Endpoint和Network。

![](attachments/Pasted%20image%2020240425165352.png)

如上图所示，它们分别是容器通信中『容器网络环境』、『容器虚拟网卡』和『主机虚拟网卡/网桥』的抽象。

**Sandbox：**
对应一个容器中的网络环境，包括相应的网卡配置、路由表、DNS配置等。CNM很形象的将它表示为网络的『沙盒』，因为这样的网络环境是随着容器的创建而创建，又随着容器销毁而不复存在的。

**Endpoint：**
实际上就是一个容器中的虚拟网卡，在容器中会显示为eth0、eth1依次类推；

**Network：**
指的是一个能够相互通信的容器网络，加入了同一个网络的容器可以直接通过对方的名字相互连接。它的实体本质上是主机上的虚拟网卡或网桥。



# 命令
## 概述
```bash
man docker-network
```

![](attachments/Pasted%20image%2020240425164544.png)

```bash
# docker network --help

Usage:  docker network COMMAND

Manage networks

Commands:
  connect     Connect a container to a network
  create      Create a network
  disconnect  Disconnect a container from a network
  inspect     Display detailed information on one or more networks
  ls          List networks
  prune       Remove all unused networks
  rm          Remove one or more networks

Run 'docker network COMMAND --help' for more information on a command.
```

```bash
docker network COMMAND --help

比如：
docker network create --help
```

![](attachments/Pasted%20image%2020240425164658.png)


```bash
man docker-network-create
```

![](attachments/Pasted%20image%2020240425164808.png)


## docker network ls
这个命令用于列出所有当前主机上或Swarm集群上的网络：

```bash
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
111b61f9844b        backend             bridge              local
50de367459f9        bridge              bridge              local
a1f6fdc7da5f        frontend            bridge              local
b8c0ed2202df        host                host                local
87aa20ea95b1        none                null                local
```

在默认情况下会看到三个网络，它们是Docker Deamon进程创建的。

它们实际上分别对应了Docker过去的三种『网络模式』：
- **bridge：**
容器使用独立网络Namespace，并连接到docker0虚拟网卡（默认模式）

- **none：**
容器没有任何网卡，适合不需要与外部通过网络通信的容器


- **host：**
容器与主机共享网络Namespace，拥有与主机相同的网络设备

在引入libnetwork后，它们不再是固定的『网络模式』了，而只是三种不同『网络插件』的实体。说它们是实体，是因为现在用户可以利用Docker的网络命令创建更多与默认网络相似的网络，每一个都是特定类型网络插件的实体。


## docker network create / docker network rm

这两个命令用于新建或删除一个容器网络，创建时可以用『--driver』参数指定使用的网络插件。

例如：
```bash
$ docker network create --driver=bridge frontend
b6942f95d04ac2f0ba7c80016eabdbce3739e4dc4abd6d3824a47348c4ef9e54
```

Docker容器可以在创建时通过『--net』参数指定所使用的网络，连接到同一个网络的容器可以直接相互通信。

当一个容器网络不再需要时，可以将它删除：
```bash
docker network rm frontend
```

### 注意
**默认的三个网络是不能被删除的**，而用户自定义的网络可以用『docker networkrm』命令删掉。



## docker network connect / docker network disconnect

这两个命令用于动态的将容器添加进一个已有网络，或将容器从网络中移除。

### 范例
我们来看一个例子。  参照前面的libnetwork容器网络模型示意图中的情形创建两个网络：
```bash
$ docker network create --subnet=192.168.10.0/24 br10
$ docker network create --subnet=192.168.20.0/24 br20
```

然后运行三个容器，让第一个容器接入frontend网络，第二个容器同时接入两个网络，三个容器只接入backend网络。首先用『–net』参数可以很容易创建出第一和第三个容器：
```bash
docker create -i --net=br10 --name=ins01 --ip=192.168.10.100 centos7.3:v1
docker create -i --net=br20 --name=ins03 --ip=192.168.20.100 centos7.3:v1

如何创建一个同时加入两个网络的容器呢？
由于创建容器时的『–net』参数只能指定一个网络名称，因此需要在创建过后再用docker network connect命令添加另一个网络：
docker create -i --net=br10 --name=ins02 --ip=192.168.10.200 centos7.3:v1
docker network connect --ip 192.168.20.200 br20 ins02


注：值得指出的是，同一主机上的每个不同网络分别拥有不同的网络地址段，因此同时属于多个网络的容器会有多个虚拟网卡和多个IP地址。
```

启动 ins01、ins02、ins03三个容器
```bash
docker start ins01 ins02 ins03
```

现在通过ping命令测试一下这几个容器之间的连通性：
```bash
 $ docker exec -it ins01 ping ins02 #可以ping通
 $ docker exec -it ins01 ping ins03 #找不到名称为ins03的容器
 $ docker exec -it ins02 ping ins01 #可以ping通
 $ docker exec -it ins02 ping ins03 #可以ping通
 $ docker exec -it ins03 ping ins01 #找不到名称为ins01的容器
 $ docker exec -it ins03 ping ins02 #可以ping通
```

这个结果也证实了在相同网络中的两个容器可以直接使用名称相互找到对方，而在不同网络中的容器直接是不能够直接通信的。

此时还可以通过docker network disconnect 动态的将指定容器从指定的网络中移除：
```bash
$ docker network disconnect br20 ins02
$ docker exec -it ins02 ping ins03 #找不到名称为ins03的容器

可见，将ins02容器实例从backend网络中移除后，它就不能直接连通ins03容器实例了。
```

##  docker network inspect

这个命令可以用来显示指定容器网络的信息，以及所有连接到这个网络中的容器列表。

![](attachments/Pasted%20image%2020240425170636.png)

# 参考
```bash
# Docker network的网络特性
http://www.hangdaowangluo.com/archives/2197


```