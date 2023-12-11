* [启程](#启程)
   * [容器生态系统](#容器生态系统)
   * [docker 镜像](#docker-镜像)
      * [base 镜像](#base-镜像)
      * [镜像的分层结构](#镜像的分层结构)
   * [构建镜像](#构建镜像)
      * [docker commit](#docker-commit)
      * [dockerfile](#dockerfile)
         * [dockerfile官方文档](#dockerfile官方文档)
      * [RUN CMD ENTRYPOINT 区别](#run-cmd-entrypoint-区别)
         * [分发镜像](#分发镜像)
   * [Docker 容器](#docker-容器)
         * [进入容器的方法](#进入容器的方法)
         * [停止容器](#停止容器)
         * [删除容器](#删除容器)
      * [资源限制](#资源限制)
         * [内存限制](#内存限制)
         * [CPU 限额](#cpu-限额)
         * [Block IO 限额](#block-io-限额)
   * [实现容器的底层技术](#实现容器的底层技术)
      * [cgroup](#cgroup)
         * [namespace](#namespace)
   * [Docker网络](#docker网络)
      * [Bridge网络](#bridge网络)
      * [user-defined 网络](#user-defined-网络)
      * [容器间通信](#容器间通信)
         * [IP通信](#ip通信)
         * [Docker DNS Server](#docker-dns-server)
         * [join容器](#join容器)
         * [与外网交互](#与外网交互)
   * [Docker存储](#docker存储)
      * [storge driver](#storge-driver)
      * [Volume](#volume)
         * [bind mount](#bind-mount)
         * [docker managed volume](#docker-managed-volume)
         * [docker和host数据共享](#docker和host数据共享)
         * [容器间数据共享](#容器间数据共享)
         * [Data-packed volume container](#data-packed-volume-container)
         * [数据迁移](#数据迁移)
         * [销毁](#销毁)
# 启程

## 容器生态系统

生态系统： 核心技术、平台技术和支持技术

容器核心技术：让Container在host运行起来的那些技术

- 容器规范

```bash
组织： Open Container Initiative (OCI)

发布两个规范： runtime spec 和 image format spec

（1） runtime 为容器提供运行环境， docker 和 runtime 的关系类比 Java 和 JVM 的 关系

		lxc runc rkt 是目前主流的三种容器 runtime

（2） 容器管理工具

	对内与 runtime 交互， 对外提供用户接口 （CLI）

（3） 容器定义工具

	docker image 容器的模版
	dockerfile 用于创建docker image
	ACI 与image类似，不过是CoreOS开发的

（5） Registry

	镜像的仓库
  有Docker Hub 和 Quay.io

（6） 容器OS

	专门用于运行容器的操作系统
```

容器平台技术：使容器作为集群在分布式环境中运行

包括容器编排引擎、容器管理平台、基于容器的PaaS

## docker 镜像

### base 镜像

含义：

- 不依赖其他镜像，从scratch构建
- 其他镜像可以以之为基础进行扩展

Linux 操作系统 由内核空间和用户空间 组成

对于base 镜像来说，底层直接用 Host 的 kernal，自己只需要提供 rootfs 就行了

base镜像用户空间和发行版一致，内核版本由Docker Host 决定，所有容器共用host的内核

### 镜像的分层结构

新镜像是从base镜像一层层叠加生成，每安装一个软件，就在现有镜像的基础上增加一层

如果有多个镜像由同一个base镜像构建而来，那docker host 只需要在内存中记载一份base镜像，就可以为所有容器提供服务

镜像的每一层都可以被共享

**可写的容器层**：当容器启动的时候，会有一个可写层加载到镜像顶部。

```bash
-----> 容器层
-----> Image 2
-----> Image 1
-----> Base Image
-----> Bootfs
```

所有改动，只会在容器层发生，镜像层只读

在容器层，用户看到的是一个叠加之后的文件系统。相同的文件上层会覆盖下层，用户只能访问到最上层的文件。（读文件的顺序从上到下）

修改和删除都是复制到容器层进行（在容器层记录修改和删除操作）

这种特性叫Copy-on-Write

## 构建镜像

- `docker commit`
- `Dockerfile`构建文件

### docker commit

步骤：运行容器 ——> 修改 ———> 保存为新镜像

```bash
1. 运行容器 

docker run -it ubuntu

2. 修改 （安装程序）

apt-get install vim 

3. 保存为新镜像

ctrl + P + Q

docker ps
docker commit NAMES Image_name
```

不是推荐的方法，且用户不知道镜像如何创建的，存在安全隐患

### dockerfile

1. 创建dockerfile
2. 进入dockerfile所在的文件夹

```bash
docker build -t ubuntu-with-vim .
```

`-t` 参数后面的是制作的镜像名，后面指定 `build context` ，docker 会默认在这里寻找dockerfile，同时为镜像构建提供所需要的文件或目录

创建过程其实也是一层层创建临时容器，然后再用 `docker commit`打包镜像

查看镜像分层结构

```bash
docker history image_name
```

docker 会进行镜像缓存，如果构建新镜像过程中，某镜像层已经存在，则不需要重新从 base 开始构建，直接用缓存构建。

- docker 镜像层是上层依赖下层的，下面某一层发生变化，上面所有的缓存都会消失

#### dockerfile官方文档

```	dockerfile
FROM  指定base镜像
MAINTAINER 作者
COPY 从build context复制到镜像
COPY src dest 
COPY ["src","dest"]
ADD 跟COPY类似，若src文件为归档文件，会被自动解压到dest
ENV 设置环境变量
EXPOSE 指定容器中的进程会监听某个端口
VOLUME 将文件或目录声明为volume
WORKDIR 设置镜像中的当前工作目录
RUN 运行指定的指令
CMD 启动时运行指定的命令 可以有多个CMD指令，但只有最后一个生效，可以被docker run之后的参数替换
ENTRYPOINT 跟CMD类似
```

### RUN CMD ENTRYPOINT 区别

run 经常用来安装软件包，CMD设置容器启动后执行的命令，ENTRYPOINT设置容器启动时运行的命令

两种传参方式：

- shell方式

```shell
RUN apt-get install python3
CMD echo "Hello World"

# 会被shell解析
```

- exec 方式

```shell
RUN ["apt-get","install","python3"]
CMD ["/bin/echo","Hello World"]

# 直接运行命令，不会被解析
```

CMD 若docker run后面有参数，将会被忽略掉

```shell
docker run -it ubuntu /bin/bash
```

ENTRYPOINT 跟CMD很像，但是不会被忽略，即使有docker run参数，也一定会被执行

且CMD可以给ENTRYPOINT提供额外的参数，但CMD提供的参数可以在容器启动时被动态替换

```shell
ENTRYPOINT ["/bin/echo","Hello"]
CMD ["World"]

# 会输出Hello World
```

替换为 docker run -it ubuntu Cha0s32、

会输出为Hello Cha0s32

#### 分发镜像

发布至官方Registry 的时候，要注意tag的使用

```shell
docker tag [镜像名] [tag]
```

```shell
docker login  -u xxx
docker tag httpd xxx/httpd:v1
docker push xxx/httpd:v1
```

搭建 私有Registry

```shell
docker run -d -p 5000:5000 -v /myregistry:/var/lib/registry registry:2
```

后面 -> 容器 映射 前面 -> 本机

```shell
# 拉取镜像，上传私有仓库
docker pull busybox
docker tag busybox:last ip:port/cha0s32/busybox:v1
docker push ip:port/cha0s32/busybox:v1
```

## Docker 容器

删除镜像

```dockerfile
docker rmi debian:latest
```

停止容器

```docker
docker stop 短ID
```

#### 进入容器的方法

- docker attach

通过ctrl + p，ctrl + q 推出attach终端

- docker exec

```shell
docker exec -it xxxxx(端ID) bash
```

退出容器使用exit就可以了



区别：exec会在容器中打开新的终端，启动新的进程，如果想在终端中查看启动命令的输出，用attach，除此之外，用exec



#### 停止容器

```shell
docker stop docker_name # 发送SIGTERM信号
docker kill docker_name # 发送SIGKILL信号
docker start

# 让容器发生错误时自动重启
docker run -d --restart=always httpd


# 只是希望容器暂停一段时间
docker pause # 不用占用CPU资源
docker unpause
```

#### 删除容器

```shell
docker rm  ID # 可一次性删除多个

# 删除所有已经退出的容器
docker rm -v $(docker ps -aq -f status=exited)
```

### 资源限制

#### 内存限制

-m 设置内存限额

--memory-swap 设置内存+swap限额

```shell
docker run -m 200M --memory-swap=300M ubuntu
```

#### CPU 限额

```shell
docker run it container_A -c 1024
```

-c 后面是比重，比如说两个容器，一个是1024，一个是512，前者使用的CPU资源将是后者的两倍

#### Block IO 限额

限制权重

```shell
docker run -it  ubuntu --name A --blkio-weight 600 
docker tun -it  ubuntu --name B --blkio-weight 300
```

限制bps和iops

```shell
--device-read-bps
--device-write-bps  # 限制读写数据量

--device-read-iops 
--device-read-iops # 限制IO次数
```

oflag=direct 可以指定用direct IO 方式写文件，只有加了这个，--device-read-bps才能生效

## 实现容器的底层技术

cgroup 实现资源限额，namespace 实现资源隔离

### cgroup

host 中存在 /sys/fs/cgroup/cpu/docker 目录

以CPU限额为例，每个容器会创建一个cgroup目录，以容器长ID命名

内存、IO限额同理

#### namespace

Mount 让容器看上去拥有整个文件系统，可以执行mount 和 unmount 命令

UTS  让容器有自己的hostname，通过docker run -h myhost 这样指定

IPC 拥有自己的共享内存和信号量

PID 拥有自己独立的一套PID

Network 拥有独立的网卡、IP、路由

User 可以独立管理自己的用户



## Docker网络

目前只讨论单个host的多docker网络

查看网络`docker network ls`

- None 忘在这个网络下的容器除了lo，没有其他任何网卡 指定 --network=none 应用场景：安全性高且不需要联网
- host 共享host的网络栈 指定 --network=host 应用场景：需求性能好，但是要牺牲灵活性，如端口冲突问题，host已经使用的端口docker容器中不能再使用
- bridge 应用更广

### Bridge网络

docker安装时会创建一个docker0的Linux bridge，如果不指定--network，容器默认都会挂到这里

容器挂到docker0的拓扑：

新建一个容器，容器内部有一个网卡eth0

会新增一个veth接口（网卡），接到docker0

eth0和veth是一对veth pair，可理解为一根虚拟网线连接起来的一堆网卡，目的就是将eth0接到docker0上



查看bridge的配置

```shell
docker network inspect bridge
```

### user-defined 网络

提供三种网络驱动：bridge、overlay、macvlan（最后两种用于创建跨主机的网络）

创建类似bridge的网络

```shell
docker network create --driver bridge my_net
brctl show # 查看网络结构变化
docker network inspect my_net # 查看my_net的配置

# 指定网段
docker network create --driver bridge -- subnet 172.22.16.0/24 --gateway 172.22.16.1 my_net2
# 创建容器时指定网络
docker run -it --network=my_net2  busybox

可通过 --ip 指定静态IP
docker run -it --network=my_net2 --ip 172.16.22.2 busybox
```



挂载在同一网络上的容器可以直接互相通信

不同网络上的容器可以通过host作为路由器通信，前提是操作系统上打开了ip forwarding

但是防火墙可能默认隔离不同的容器network

解决方法：给一个网络加上一块另一个网络的网卡

```shell
docker network connect my_net2 容器短ID
```



### 容器间通信

#### IP通信

详见上边说的，加入同一个网络或者添加一张一个网络的网卡，这里不加赘述



#### Docker DNS Server

上一种方法要知道对应的IP，如果无法确认IP，可以使用该方法，可通过容器名通信

如：在docker run的时候挂在同一个网络上，可以直接ping docker-name 的方法访问到

限制：只能在user-defined网络中能使用



#### join容器

可以使多个容器共享一个网络栈

```shell
# 创建一个web1的httpd容器
docker run -d -it --name=web1 httpd

# 创建一个busybox容器，指定joined容器为web1
docker run -it --network-container:web1 busybox
```

此时busybox的网卡mac地址和IP完全跟web1一致

可以直接使用127.0.0.1来访问web1的http服务



#### 与外网交互

访问外网：

默认创建的容器通过nat访问外网

具体过程是：docker0收到容器172.16.0.2 的访问外网的流量，随后丢给NAT处理，NAT将源地址转化为docker host的网卡地址，再把流量转发给目的地址

外网访问docker：

通过端口映射实现

docker将容器对外提供服务的端口映射到host的某个端口

```shell
docker run -d -p host_port:docker_port httpd
```

每一个映射的端口，host都会启动一个docker-proxy进程来处理访问





## Docker存储

### storge driver

分层结构的基础

实现了多层数据的堆叠并为用户提供一个单一的合并之后的统一视图

有多种storge driver，目前解决方案是针对每一个Linux发行版，在安装docker的时候，会指定默认storge driver还有底层文件系统

应用场景：无状态的容器，数据不需要持久化存储，容器删除的时候数据可以一起删除



### Volume

本质上是Docker Host 文件系统中的目录或文件，能够直接被mount到容器的文件系统中

数据可以永久保存，即使容器被销毁

volume的容量取决于文件系统当前未使用的空间，目前没有方法设置volume的容量

#### bind mount

将host上已存在的目录或文件mount到容器上

使用 -v ， -v <host path>:<container path>

bind mount 可以让host 与 容器共享数据

可以指定数据的读写权限，默认可读可写

指定只读 ，容器无法对数据进行修改

```shell
-v ~/htdocs:/usr/local/apache2/htdocs:ro
```

缺点：当容器迁移到其他host，该host没有要mount的数据或数据不在相同的路径，操作会失败



#### docker managed volume

不需要指定mount源，知名mount point 就行

```shell
docker run -d -p 80:80 -v /usr/local/apache2/htdocs httpd
```

Data volume 的位置在配置文件中可以看到

```shell
docker inspect 容器ID
```

关注Mounts列表，Source就是该volume在host上的目录

步骤：

docker在 /var/lib/docker/volumes 中生成一个随机目录作为mount源

如果mount到的目录已经存在，则把数据复制到mount源

将volume mount到相关目录



#### docker和host数据共享

docker managed volume由于是容器启动时才生成，所以需要将共享数据复制到volume中

一种就是直接linux cp命令将数据复制到mount源文件夹

一种是通过docker cp在容器和host间进行数据复制

```shell
docker cp ~/htdocs/index.html 容器ID:/usr/local/apache/htdocs/
```



#### 容器间数据共享

一种方法，一个目录mount到不同容器的文件系统

另一种方法是使用 volume container

volume就是专门给其他容器提供volume的容器

```shell
# 建立 volume container

docker create --name vc_data \
-v ~/htdocs:/usr/local/apache2/htdocs \
-v /other/useful/tools \
busybox
```

建立就好，本来不需要运行，只是提供数据

bind mount：存放Web Server的静态文件

docker managed vulume：存放一些实用工具

其他容器可以通过 --volumes-from 实用vc_data 这个volume container

```shell
docker run --name web1 -d -p 80 --volume-from vc_data httpd
```



#### Data-packed volume container

上述例子中，volume container 的数据是在host中，现在目的是将数据完全放到volume container中

原理：将数据打包到镜像中

```dockerfile
FROM busybox:latest
ADD htdocs /usr/local/apache2/htdocs
VOLUME /usr/local/apache2/htdocs
```

打包镜像，使用该镜像创建容器

启动httpd容器，并使用--volume-from指定volume container

```shell
docker run -d -p 80:80 --volume-from vc_data httpd
```

该方法非常适合迁移



#### 数据迁移

以docker私有仓库为例

创建命令

```shell
docker run -d -p 5000:5000 -v /myregistry:/var/lib/registry registry:2
```

迁移步骤（启用新版本）

1. docker stop 掉当前容器
2. 启用新版本容器并mount原有的volume

```shell
docker run -d -p 5000:5000 -v /myregistry:/var/lib/registry registry:latest
```



#### 销毁

对于docker managed volume的，在docker rm容器的时候，带上-v 参数，会把volume一并删除

前提是没有其他容器mount该volume

没有容器mount到volume，但volume还存在，可以使用

```shell
docker volume ls # 查询
docker volume rm {volume_ID} #删除

# 批量删除
docker volume rm $(docker volume ls -q)
```



