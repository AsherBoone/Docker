# docker 1.9版本新网络特性介绍---overlay

简介： 在docker 1.9版本中引入了一整套的自定义网络和跨主机的网络支持。

## libnetwork和Docker网络

libnetwork项目从lincontainer和docker代码的分离早在docker1.7版本就已经完成了抽离。

libnetwork所做的最黑小囡的事情是定义了一组标准的网络模型(container network model，简称CNM)，只要符合这个网络接口就能被用于容器之间通信，而通信的过程和细节可以完全由网络接口来实现。

这个网络模型中定义了三个术语：Sandbox、Eedpoint、Network

![docker-network](http://gogs.gyyx.cn/chenjiayun/Docker/raw/master/picture/docker-network.png)

如上图所示，它们分别是网络通信中 容器网络环境、容器虚拟网卡、主机虚拟网卡/网桥的抽象。

- Sandbox：对应一个容器中的网络环境，包括相应的网卡配置、路由表、DNS配置等。CNM很形象的将它表示为网络的『沙盒』，因为这样的网络环境是随着容器的创建而创建，又随着容器销毁而不复存在的

- Endpoint：实际上就是一个容器中的虚拟网卡，在容器中会显示为eth0、eth1依次类推

- Network：指的是一个能够相互通信的容器网络，加入了同一个网络的容器直接可以直接通过对方的名字相互连接。它的实体本质上是主机上的虚拟网卡或网桥。

## docker network

1. docker network ls 列出当前主机上的或swarm 集群上的网络
	- 默认情况会看到三个网络，它们是docker deamon进程创建的。它们实际上分别对应了docker过去的三种 网络模式：
		* bridge:容器使用独立网络Namespace，并连接到docker0虚拟网卡（default）
		* none：容器没有任何网卡，适合不需要与外部通过网络通信的容器。
		* host:容器与主机共享网络Namespace，拥有与主机相同的网络设备

	- 在引入libnetwork后，它们不再是固定的『网络模式』了，而只是三种不同『网络插件』的实体。说它们是实体，是因为现在用户可以利用Docker的网络命令创建更多与默认网络相似的网络，每一个都是特定类型网络插件的实体。

2. docker network create /docker netowrk rm

    - 这两个命令用于新建和删除一个容器网络，创建时可以用 --driver参数使用网络插件
	
		* docker network create --driver=bridge br0
		* docker 容器可以在创建时通过 --net来指定使用的网络。
		* docker network rm br0
		* 若要同时加入两个或多个网络可以用dockert networkconnect br0 web2
		* 不同网络间容器是无法联通的。
		* docker network disconnect br0 web2 断开容器网络连接
		* docker network inspect 查看网络详细信息。

	- 同一主机上的每个不同网络分别拥有不同的网络地址段，因此同时属于多个网络的容器会有多个虚拟网卡和多个IP地址。

	- libnetwork带来的最直观变化实际上是：docker0不再是唯一的容器网络了，用户可以创建任意多个与docker0相似的网络来隔离容器之间的通信。然而，要仔细来说，用户自定义的网络和默认网络还是有不一样的地方
		* 默认的三个网络是不能被删除的，而用户自定义的网络可以用『docker network rm』命令删掉
		* 连接到默认的bridge网络连接的容器需要明确的在启动时使用『–link』参数相互指定，才能在容器里使用容器名称连接到对方。而连接到自定义网络的容器，不需要任何配置就可以直接使用容器名连接到任何一个属于同一网络中的容器。这样的设计即方便了容器之间进行通信，又能够有效限制通信范围，增加网络安全性
		* 在Docker 1.9文档中已经明确指出，不再推荐容器使用默认的bridge网卡，它的存在仅仅是为了兼容早期设计。而容器间的『–link』通信方式也已经被标记为『过时的』功能，并可能会在将来的某个版本中被彻底移除

## Docker的内置overlay网络
	
内置跨主机的网络通信一直是Docker备受期待的功能，在1.9版本之前，社区中就已经有许多第三方的工具或方法尝试解决这个问题，例如Macvlan、Pipework、Flannel、Weave等。虽然这些方案在实现细节上存在很多差异，但其思路无非分为两种：二层VLAN网络和Overlay网络。

1. 创建overlay网络

实现跨主机网络连通，需要docker所有节点配置--cluster-store 和--cluster-advertise 来指定所使用的配置存储服务地址。并且在每个节点都应该有一个名称、ID和属性完全一致的网络，它们之间还要相互认可对方为自己在不同节点的副本。如何实现这种效果呢？目前的Docker network命令还无法做到，因此只能借助于Swarm
 
–cluster-store=etcd://<Etcd所在主机IP>:2379/store-–cluster-advertise=eth1:2376
 
	docker network create --driver overlay --subnet=192.168.200.0/24  multihost  

	docker network create --driver overlay --subnet=10.11.0.0/24 --gateway=10.11.0.254 multi-host-network

2. 跨主机创建容器

	docker -H tcp://<Master节点地址>:3375 run-td –name ins01 –net ovr0 –env=”constraint:node==swarm-agent-1″ nginx

	docker -H tcp://<Master节点地址>:3375 run-td –name ins02 –net ovr0 –env=”constraint:node==swarm-agent-2″ ubuntu
3. 连通性测试

	docker -H tcp://<Master节点地址>:3375 exec-it ins01 ping ins02
	
	连通正常
