# docker入门
简介：Docker是PaaS提供商dotCloud开源的一个基于LXC的高级容器引擎，Docker基于Go语言开发，自2013年以来非常火热，源代码托管在github上，基于go语言并遵从Apache2.0协议开源；
基于Docker的沙箱环境可以实现轻型隔离，多个容器间不会相互影响；Docker可以自动化打包和部署任何应用，方便地创建一个轻量级私有PaaS云，也可以用于搭建开发测试环境以及部署可扩展的web应用等。

## docker和虚拟化的区别

简单来说，docker就是一个应用程序执行容器，类似虚拟机的概念。但是与虚拟化技术有区别：

1. 虚拟机技术依赖物理cpu和内存，是硬件级别的；而docker构建在操作系统上，利用操作系统的containerzation技术，所以docker可以在虚拟机上运行。
2. 虚拟化系统一般都是值操作系统镜像，比较复杂，成为“系统”；而docker开源而且轻量，成为“容器”，单个容器适合部署少量应用，比如一个redis、一个memcached。
3. 传统的虚拟化技术使用快照来保存状态；而docker在保存状态上不仅更为轻便和低成本，而且引入了类似源代码管理机制，将容器的快照历史版本一一记录，切换成本很低。
4. 传统的虚拟化技术在构建系统的时候较为复杂，需要大量的人力；而docker可以通过Dockfile来构建整个容器，重启和构建速度很快。更重要的是Dockfile可以手动编写，这样应用程序开发人员可以通过发布Dockfile来指导系统环境和依赖，这样对于持续交付十分有利。
5. Dockerfile可以基于已经构建好的容器镜像，创建新容器。Dockerfile可以通过社区分享和下载，有利于该技术的推广。

## docker主要特性

1. 文件系统隔离：每个进程容器运行在完全独立的根文件系统里
2. 资源隔离：可以使用cgroup为每个进程容器分配不同的系统资源，例如cpu和内存
3. 网络隔离：每个进程容器运行在自己的网络命名空间里，拥有自己的虚拟接口和ip地址。
4. 写时复制：采用写时复制方式创建跟文件系统，这让部署变得极其快捷，并且节省内存和硬盘空间
5. 日志记录：docker将会收集和记录每个进程容器的标准流（stdout/stderr/stdin），用于实时检索或批量检索
6. 变更管理：容器文件系统的变更可以提交到新的映像中，并可重复使用以创建更多的容器。无需使用模板或手动配置。
7. 交互式Shell：Docker可以分配一个虚拟终端并关联到任何容器的标准输入上，例如运行一个一次性交互shell。
 
 	相关连接: 
 	
 	<https://www.infoq.com/news/2013/03/Docker>
 	<http://www.cnblogs.com/Bozh/p/3958469.html>
 	
## docker 安装
 
	yum -y install docker  #centos 7 系统安装，官方称在最好在3.8以上的内核上安装
	
	#另安装极为简单，其他系统安装类似，配置文件/etc/sysconfig/docker
	
	systemctl enable docker.server;systemctl start docker.server
	
## docker 镜像下载

默认连接到docker官方仓库去下载镜像，但GFW被墙。如果无法翻墙下载，建议用国内的加速器，例如daocloud、阿里等，亲测daocloud下载很快。

	docker search centos #搜索centos相关镜像
	docker pull kz8s/centos #下载镜像
	
daocloud 加速器url ： <https://dashboard.daocloud.io>

docker 官方仓库（docker hub）： <https://hub.docker.com/explore/> #需要注册登录

## docker 目录介绍

* docker配置文件路径为：/etc/sysconfig/docker
* docker数据文件路径：/var/lib/docker

	* /var/lib/docker/repositories-devicemapper #镜像的info文件
	* /var/lib/docker/graph/ #存放镜像id，其下有镜像元数据，layer大小，rootfs该容器镜像
	* /var/lib/docker/devicemapper/ #保存容器的data 、metadata

## docker常规命令介绍

* docker info #查看docker系统信息；包括数据目录，版本，资源，镜像，容器数量等。
* docker version #查看版本
* docker --help #查看帮助
* **
* docker search centos #从docker hub搜索centos镜像
	
	`docker search -s 40 --automated --no-trunc django`
	 
	-s 40:收藏数量不小于40，--automated：自动化构建类型，--no-trunc：显示完整的镜像描述
* docker pull IMAGE_NAME #从仓库拉取或更新指定的镜像； -a :拉取所有指定名称镜像

	`docker pull million12/nginx-php` 
	
* docker login ：登陆到docker hub，可以从自己在docker hub建立的私有仓库从拉取或push
* docker logout：退出docker hub
* docker images：显示本地所有仓库
	
	`docker images ubuntu` #显示指定镜像
	
	`docker images -q` #仅列出镜像ID；-a 列出所有镜像。
* docker run ：启动一个容器

	`docker run -d -it --restart=always --name CJY_TEST -p 8080:80 -v /data:/var/www/html centos`
	
	-d 后台运行容器，并返回容器ID；

	-i 以交互模式运行容器，通常与 -t 同时使用
	
	-t 为容器重新分配一个伪输入终端，通常与 -i 同时使用
	
	--name 指定名称
	
	--restart 保持启动状态，当宿主机重启后能自动启动
	
	-e 设置环境变量

	-p 端口映射，支持的格式为：ip:hostPort:containerPort | ip::containerPort | hostPort:containerPort
	
	更多参数，请help查看。
* docker attach：可以进入到一个已经运行的容器stdin，但从这个stdin退出的时候会停止容器

	`docker attach 1234de3dde` #退出按ctrl + p + q可以防止容器停止

* docker exec：虚拟出一个伪终端进行操作，但退出不会影响容器状态

	`docker exec -it ed32d23d1 /bin/bash` #exit不影响。
	
* docker ps ：列出所有运行的容器

	`docker ps -a` #列出所有容器
	
	`docker -l` #列出最新创建的容器，-n=4：最近创建的4个容器；--no-trunc（完整ID）,-q（ID）,-s:显示容器大小
	
* docker rm/rmi ：删除容器/镜像 （关机/没有被使用）

	`docker rm 5433cd3d2` #删除已经关机容器；rmi：删除未被使用的镜像； -f：强行删除
	
* docker stop/start/restart Container：开启/关闭/重启一个或多个指定容器

	-a：待完成
	
	-i：启动一个容器并进入交互模式
	
	-t 10：停止或者重启的超时时间，超时后系统将杀死进程

* docker kill -s "KILL" Container_NAME ；杀死一个或多个指定容器进程,KILL:信号
* docker save ：将指定镜像保存成tar归档文件
* docker export/import ：容器数据的导入导出
* docker top ID：查看一个正在运行容器的进程，支持ps参数。
* docker inspect：查看镜像、容器的详细参数，默认为json格式，-f 指定返回的模板文件。
* docker pause/unpause ID：暂停/开启某一容器的所有进程。
* docker tag：本地镜像打tag，然后push到仓库。
* docker logs ID：获取容器运行时的输出日志。 -f跟踪容器日志的最近更新 -t时间戳 --tail=“10”显示最新10条数据。

使用：
	
   gogs不支持图片连接播放：请点击 [播放地址](https://asciinema.org/a/e08ool01qp5agq9v2xxgzoliy)

[![asciicast](https://asciinema.org/a/e08ool01qp5agq9v2xxgzoliy.png "Demo")](https://asciinema.org/a/e08ool01qp5agq9v2xxgzoliy)
