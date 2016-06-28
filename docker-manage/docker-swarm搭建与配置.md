# swarm搭建、安装、配置
## 一、swarm简介

Swarm是Docker公司在2014年12月初新发布的容器管理工具。和Swarm一起发布的Docker管理工具还有Machine以及Compose。
Swarm是一套较为简单的工具，用以管理Docker集群，使得Docker集群暴露给用户时相当于一个虚拟的整体。Swarm使用标准的Docker API接口作为其前端访问入口，换言之，各种形式的Docker Client(dockerclient in go, docker_py, docker等)均可以直接与Swarm通信

## 二、架构

![swarm icon](http://gogs.gyyx.cn/chenjiayun/Docker/raw/master/picture/swarmarchitecture.jpg)

介绍：

1. Swarm架构中中最主要的处理部分是Swarm节点。管理对象是Docker cluster。Docker cluster由多个Docker Node组成，负责给swarm发送请求的是docker client。
2.	五种发现方式
	* 节点发现
	
	`swarm manage --discovery dockerhost01:2375,dockerhost02:2375,docker03:2375 -h=0.0.0.0:2375` 
	* 文件发现
	
	文件发现利用防止在文件系统中的配置文件，使用ip：port的格式来列出集群中的docker主机，
	
	`swarm manage --discovery file://etc/swarm/cluster_config -h=0.0.0.0:2375`
	* consul发现
	
	支持consul发现，利用consul的键值存储，来维护集群构成集群服务器的ip：prot列表。每个docker主机会以join模式运行一个swarm守护进程（下文讲到），并指向consul集群的http接口（进行服务注册）。
	
	`swarm join --discovery consul://192.168.30.12:4040/swarm --addr=192.168.30.5:2375`
	* etcd发现
	
	etcd发现的工作方式与consul发现大致相同，etcd可以通过心跳检测来维护一个记录集群中活跃服务器列表(active servers)：
	
	`swarm join --discovery etcd://192.168.30.11:4040/swarm --addr=192.168.30.9:2375`
	* zookeeper发现
	
	zookeeper发现模式与其他基于键值对存储的配置模式类似，一个ZK集合被创建之后用于保存主机信息列表同时还有一个与Docker一起运行的客户端，这个客户端可以向键/值存储服务发送心跳检测信息，几乎可以实时地维护列表。Swarm master也会连接到ZK集合并使用 /swarm 路径下的的信息来维护主机列表。
	`swarm join --discovery zk://192.168.30.15:4040/swarm --addr=192.168.30.6:2375`

3. 两种调度

	当开启一个docker容器时，swarm将基于过滤器选择一组服务器，然后根据调度机制来分配每次运行的命令。过滤器会告诉Swarm哪些服务器上的容器是可用的或是不可用的，之后调度器会把命令分发到可用的服务器上。
	* Affinity
	* port
	* Healthy
	* Random
	* Binpacking
	
相关连接<http://technolo-g.com/intro-to-docker-swarm-pt2-config-options-requirements/>

## 安装与配置

1. 下载swarm
	
	官方推荐用镜像的方式安装部署，在集群所有节点需要执行：

	`docker pull swarm`
2. 修改配置文件
   
   在所有节点修改配置文件/etc/sysconfig/docker
   `OPTIONS='-H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --insecure-registry 192.168.30.8:5000 --selinux-enabled'`
   
   `systemctl daemon-reload`
   
   `systemctl restart docker.service`
	
3. 安装etcd（k/v存储,eg: ip=192.168.30.11）
	
	1) 安装：`yum -y install etcd`
	2）配置：`cat /etc/etcd/etcd.conf | egrep "^#|^$"`
	
	ETCD_NAME=default
	ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
	ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
	ETCD_ADVERTISE_CLIENT_URLS="http://192.168.30.11:2379"
	
	`systemctl enable etcd.service`
	
	`systemctl start etcd.service`
4. 将所有机器加入集群，在节点分别执行：

	`docker run -d --restart=always swarm join --addr=192.168.30.8:2375 etcd://192.168.30.11/swarm`
	
	`docker run -d --restart=always swarm join --addr=192.168.30.9:2375 etcd://192.168.30.11/swarm`
	
	......
	
5. 查看集群节点列表(任意一台机器)
	
	`docker run --rm swarm list etcd://192.168.30.11/swarm`

	```
[#1#root@cjy-docker-master ~]$docker run --rm swarm list etcd://192.168.30.11:2379/swarm
192.168.30.10:2375
192.168.30.11:2375
192.168.30.8:2375
192.168.30.9:2375
	```	

6. 选择swarm manage节点，在其上运行(eg:ip=192.168.30.8)：
	
	`docker run -d --restart=always -p 2376:2375 swarm manage -H=0.0.0.0:2375 etcd://192.168.30.11:2379`
	
7. 查看swarm集群信息

	`export DOCKER_HOST=192.168.30.8`
	
	`docker info`
	
	```
	[#198#root@cjy-docker-master ~]$docker info
Containers: 6
Images: 11
Server Version: swarm/1.2.2
Role: primary
Strategy: spread
Filters: health, port, containerslots, dependency, affinity, constraint
Nodes: 3
 cjy-docker-master.novalocal: 192.168.30.8:2375
  └ ID: OTCX:F2ZL:HQS3:MK5O:UVUB:LQDT:WMB2:D3ZT:5N7O:OKJG:5U3K:Y44H
  └ Status: Healthy
  └ Containers: 3
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.019 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=3.10.0-123.el7.x86_64, operatingsystem=CentOS Linux 7 (Core), storagedriver=devicemapper
  └ Error: (none)
  └ UpdatedAt: 2016-05-19T09:20:13Z
  └ ServerVersion: 1.9.1
 cjy-docker-node1.novalocal: 192.168.30.9:2375
  └ ID: 22GX:SJSZ:LQA4:YBCW:WJZG:PJ24:IOBY:4OZN:MQKO:7HZQ:RS7M:AD5V
  └ Status: Healthy
  └ Containers: 2
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.019 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=3.10.0-123.el7.x86_64, operatingsystem=CentOS Linux 7 (Core), storagedriver=devicemapper
  └ Error: (none)
  └ UpdatedAt: 2016-05-19T09:20:23Z
  └ ServerVersion: 1.9.1
 cjy-docker-node2.novalocal: 192.168.30.10:2375
  └ ID: DPOT:4Y2X:J2WH:Z45V:L26I:EKPT:DDXE:YTZP:U32Z:LAOM:ORUF:BCTK
  └ Status: Healthy
  └ Containers: 1
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.019 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=3.10.0-123.el7.x86_64, operatingsystem=CentOS Linux 7 (Core), storagedriver=devicemapper
  └ Error: (none)
  └ UpdatedAt: 2016-05-19T09:20:54Z
  └ ServerVersion: 1.9.1
Kernel Version: 3.10.0-123.el7.x86_64
Operating System: linux
CPUs: 3
Total Memory: 3.058 GiB
Name: e4719274b54d
	```
	
8. 操作集群主机

	`docker ps -a` #列出集群所有的容器
	
	`docker images` #列出集群所有镜像
	
	`docker stop/rm ad112344` #停止或删除集群某一container
	
	`docker run -d --name cjy_test --restart=always -p 4433:443 --shm-size=64m 192.168.30.8:5000/nginx` #创建容器，shm指定shm分区大小为64，可以-m指定内存大小
	
9. swarm manage高可用

	1. 当主manage 宕机后，整个集群都没法管理。做了高可用后，可随时进行切换。
	2. 修改上面的第6步
		`docker run -d --restart=always -p 2376:2375 swarm manage -H=0.0.0.0:2375 --replication --advertise 192.168.30.8:2376 etcd://192.168.30.11:2379/swarm`
	3. 另选一台为swarm备机（replica），可以有多个replica。在swarm manage2节点：
	
	`docker run -d -p 2376:2375 swarm manage -H=0.0.0.0:2375  --replication --advertise 192.168.30.9:2376  etcd://192.168.30.11:2379/swarm`
	
	4. 查看集群
		
		```
		[#3#root@cjy-docker-master ~]$
[#3#root@cjy-docker-master ~]$docker info
Containers: 11
Images: 17
Server Version: swarm/1.2.2
Role: replica
Primary: 192.168.30.11:2376
Strategy: spread
Filters: health, port, containerslots, dependency, affinity, constraint
Nodes: 4
 cjy-docker-master.novalocal: 192.168.30.8:2375
  └ ID: OTCX:F2ZL:HQS3:MK5O:UVUB:LQDT:WMB2:D3ZT:5N7O:OKJG:5U3K:Y44H
  └ Status: Healthy
  └ Containers: 3
		```
	5. 当主swarm manage挂了后会重新选举master，如上“Role： replica”
	
## docker compose
简介：Compose可以让用户在集群中部署分布式应用。简而言之，docker compose属于一个“应用层”的服务，用来定义和运行一个或多个容器应用的公寓。使用compose可以简化容器镜像的建立及容器的运行。定义容器间如何关联。
compose是使用yml文件来定义多容器应用的，它还会用`docker-compose up`命令吧完整的应用运行起来。

### 架构

![compose icon](http://gogs.gyyx.cn/chenjiayun/Docker/raw/master/picture/docker-compose.png)

#### compose使用
compose基本上遵循以下三步：
	
1. 用dockerfile文件定义应用的运行环境，以便应用在任何地方都可以复制；基于这个dockerfile，可以构建出一个docker镜像。
2. 用docker-compose.yml文件定义应用的各个服务，以便这些服务何以作为此应用的组件一起运行。
3. 最后，执行docker-compose up命令，这样compose就会创建和运行整个应用。
#### compose安装
官方推荐安装方式：
<https://docs.docker.com/compose/install/>
	
#### 其他功能方式
<https://docs.docker.com/compose/>	
	
### docker machine	

简介：是一个简化docker安装的命令行工具，通过一个简单的命令行即可在相应的平台上安装docker，为用户提供灵活的功能，使得用户可以在任意主机上运行docker容器，而且还可以控制machine创建docker主机。简而言之，docker machine就是一个docker host主机和经过配置的docker client结合体。

<https://docs.docker.com/machine/overview/>
	
