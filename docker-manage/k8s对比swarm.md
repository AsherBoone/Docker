# swarn、kubernetes对比

## kubernetes
Kubernetes是Google开源的容器集群管理系统。它构建于docker技术之上，为容器化的应用提供资源调度、部署运行、服务发现、扩容缩容等整一套功能，本质上可看作是基于容器技术的mini-PaaS平台。

## 总体概览

![k8s](http://www.xinbujing.com/images/art/20150609/143379900700.jpg)

操作对象

1. pod：最小的调度单元，包含一个或多个container，把耦合度很高的一类容器组成一起，比如前端、后端、数据库可以创建三个pod。

2. Service：是pod路由代理的抽象，用于解决pod之间服务发现的问题，保证pod的动态变化依然能够根据service的ip能够定位到后端主机(service能提供代理，并行自己会有一个ip)。每一个服务后面都有很多对应的容器来支持，通过Proxy的port和服务selector决定服务请求传递给后端提供服务的容器，对外表现为一个单一访问地址，外部不需要了解后端如何运行

3. Replication controller：是pod的复制抽象，实现复制多个pod副本，一般一个应用有多个pod来支撑，并且能保证复制的副本数，当意外缺少是，会及时创建。
4. Proxy：解决了同一宿主机相同服务端口冲突的问题，还提供了Service转发服务器端口对外提供服务的能力，proxy后端使用了随机、轮询负载均衡算法

## 整体架构

![k8s](http://images.cnitblog.com/blog/23202/201409/252105378731272.png)

介绍：

master 运行的三个组件

1. APIserver：作为kubernetes系统的入口，封装了核心对象的增删改查操作，以RESTFul接口方式提供给外部客户和内部组件调用。它维护的REST对象将持久化到etcd（一个分布式强一致性的key/value存储）。

2. schedule：负责集群的资源调度，为新建的pod分配机器。

3. controller-manager：负责执行各种控制器：

	1. endpoint-manage：定期关联service和pod，保证service到pod的映射总是最新的。

	2. replication-controller定期关联	replicationcontroller和pod，保证rc的定义的复制数量与实际运行的pod数量总是一直的

	

slave（成为monitor）运行的两个组件：

1. kubelet：负责管控docker容器，如启动/停止、监控运行状态等，它会定期从etcd获取分配到的本机的pod，并根据 	pod信息启动或停止相应的容器。同时，它也会接受apiserver的http请求，汇报pod的运行状态。

2. proxy：负责为pod提供代理。它会定期从etcd获取所有的service，并根据service的信息创建代理。当某个客户pod要访问其他pod时，访问请求会经过本机proxy做转发。

## kubernetes优缺点

### 优点

1. 资源管理，可以对一组pod进行资源控制，（内存cpu）

2. 一组pod共享挂载卷，通过模板定义。

3. flannel实现集群网络通信。

4. 服务发现。一组pod提供一个service ip，代理访问。

5. 监控，有一套监控体系，可以采集容器的基本信息监控。

6. 负载均衡，rc提供不同主机多个pod副本，可实现。

7. 部署运行，可以对容器进行开关机、部署新应用等。

8. 支持dns、分布式存储、监控、日志等全套方案

9. 强大的自定义能力

### 缺点

1. kubernetes使用了一个完全不同的docker CLI（命令行接口。），需要看官方文档，且有大量命令需要学习。

2. 使用自己的api来间接性的操作docker 。

3. 使用了自己格式的yaml来配置定义。（pod，service，rc），需要学习语法。

## swarm

swarm是一个原生的docker集群管理工具，是有docker公司研发，swarm使用标准的api，所以容器能够使用docker run等命令启动与管理。

## 架构

![swarm](http://static.open-open.com/news/uploadImg/20150909/20150909230258_690.jpg)

![swarm](http://gogs.gyyx.cn/chenjiayun/Docker/raw/master/picture/swarmarchitecture.jpg)

介绍

1. docker api用于管理镜像的生命周期
2. swarm CLI用于集群管理
3. leadership:提供集群的HA，防止单点故障。
4. discovery service是swarm的发现服务，他会在每个node中注册一个agent将各个节点的ip端口上报，manager会从发现服务中读取各节点信息。五种发现方式：
	1. Node Discovery
	2. File discovery
	3. Consul discovery
	4. Etcd discovery
	5. Zookeeper discovery
	
5. Schedule 调度模块，用于容器调度时选择最优节点。分两步：
	1. filter(过滤)，当创建或运行容器时，会告诉调度器那些节点是可用的（符合要求的）。Filter分两类：节点过滤和基于容器的配置过滤
	2. strategy根据策略选择最优节点。

## swarm优缺点

### swarm优点

1. 官方提供原生的docker管理工具，基于docker api
2. 架构和逻辑简单，安装直接、灵活，操作容易。
3. Docker 原生态支持持久性卷组，软件解决了网络问题
4. 部署运行可以使用docker原生命令
5. 服务发现，五种发现方式
6. 支持故障切换，检测到主机异常或宕机会自动启动
7. 可以用compose实现多主机负载均衡，需要编写和维护yml文件

### swarm缺点

1. 如果做一些不被swarm api支持的操作，无法执行
2. 对复杂的调度支持的不好，比如：以定制接口方式定义的调度。
3. 能在manager上实现HA，但容器异常后不能主动新建
4. 没有集成监控，需要用其他工具来实现。

## kubernetes、swarm综合对比

1. k8s使用自己的命令来操作集群，用自定义的yaml/json来对pod svc rc操作，不支持docker CLI ，swarm基于docker api原生态支持docker CLI。
2. k8s自身支持dns、日志收集 、监控UI，而swarm不支持这些，需要用第三方工具实现。
3. k8s支持服务发现，调度；swarm支持五种发现方式、多种调度策略，用户自定义强。
4. k8s支持容器负载均衡，故障切换；swarm只能在manage做ha，在容器级别做负载均衡
5. k8s可以实现docker api以外的功能需求（功能强大，扩展性高），swarm只能支持docker api所支持的功能。
6. k8s安装、使用、维护、管理集群较为复杂，swarm则比较容易。

***
建议：
	如果满足一般的使用、简单的日常管理，对功能没有特殊的需求建议用swarm，但是没有容器容灾。
	如果在使用、管理、功能和扩展方便要求较高的话，建议用kubernetes。

注：注：mesos相比kubernetes是个重量级docker集群管理框架，有很好的资源管理机制；kubernetes+mesos 、swarm+mesos是很好的实现方式。
