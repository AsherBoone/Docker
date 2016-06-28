# RedHat 7 下配置 NFS + convoy作为docker存储

简介：NFS为网络文件系统(Network file system)的缩写,由Sun公司开发,通过网络,NFS支持在不同的文件系统之间共享文件.用户不必关心计算机的型号,以及使用的操作系统,如果想使 用远程计算机上的文件,只要用mount命令将远程的目录挂接在本地文件系统下,就可以如同使用本地文件一样使用这个资源.

## 安装NFS服务器

1. yum -y install nfs-utils  
2. 编辑配置nfs配置文件（可选）

	LOCKD_TCPPORT=30001 #TCP锁使用端口 
    
	LOCKD_UDPPORT=30002 #UDP锁使用端口
    	
	MOUNTD_PORT=30003 #挂载使用端口
    
	STATD_PORT=30004 #状态使用端口
	
	改好以后保存退出，除了以上四个端口要通过iptables，还有nfs协议端口2049以及rpc的111端口，这样才能顺利的使用nfs服务
3. 服务器设置nfs共享，vim /etc/exports

	/data/web_data 192.168.10.0/24(rw,sync,no_root_squash,no_subtree_check)

	
	配置文件的格式为：

	[共享目录] [主机名或IP(参数,参数)]

	共享目录：服务器上需要共享的目录路径；

	主机名或IP：如果主机名或IP地址为空，则表示共享给所有客户机；

	参数：NFS共享的常用参数如下：
    
	ro：只读
    
	rw：读写
    
	sync：同步写入资料到内存与硬盘中
    
	async：资料会先暂存于内存中，而非直接写入硬盘
    
	secure：NFS通过1024以下的安全TCP/IP端口发送
    
	insecure：NFS通过1024以上的端口发送
    
	wdelay：如果多个用户要写入NFS目录，则归组写入（默认）
    
	no_wdelay：如果多个用户要写入NFS目录，则立即写入，当使用async时，无需此设置。
    
	hide：在NFS共享目录中不共享其子目录
    
	no_hide：共享NFS目录的子目录
    
	subtree_check：如果共享/usr/bin之类的子目录时，强制NFS检查父目录的权限（默认）
    
	no_subtree_check：同上，但不检查父目录权限
    
	all_squash：共享文件的UID和GID映射匿名用户anonymous，适合公用目录。
    
	no_all_squash：保留共享文件的UID和GID（默认）
    
	root_squash：root用户的所有请求映射成如anonymous用户一样的权限（默认）
    
	no_root_squash：root用户具有根目录的完全管理访问权限
    	
	anonuid=xxx：指定NFS服务器/etc/passwd文件中匿名用户的UID
    
	anongid=xxx：指定NFS服务器/etc/passwd文件中匿名用户的GID

	当将同一目录共享给多个客户机，但对每个客户机提供的权限不同时，可以这样：

		[共享目录] [主机名1或IP1(参数1,参数2)] [主机名2或IP2(参数3,参数4)]
		示例 ：  
		cat /etc/exports
 		/server 192.168.10.10(rw,no_root_squash) *(ro)

		共享目录/server 允许192.168.10.10客户机读写并且root用户有管理权限。其他机器只有可读权限。
4. 启动

	systemctl enable rpcbind
	
	systemctl enable nfs-server

	systemctl start rpcbind nfs-server

5. 修改共享目录

	如果修改了/etc/exports文件后不需要重新激活nfs，只要使用exportfs命令重新扫描一次/etc/exports文件，且重新将设定加载即可。

    	exportfs –arv
	exportfs命令用法：
    	
		exportfs [-aruv]

	参数说明如下：
    -a：全部挂载（或卸载）/etc/exports文件内的设定。
    -r：重新挂载/etc/exports中的设置，此外同步更新/etc/exports及/var/lib/nfs/xtab中的内容。
    -u：卸载某一目录。
    -v：在export时将共享的目录显示在屏幕上。
 

	确认NFS成功运行：
	rpcinfo -p | grep nfs

	    100003    2   udp   2049  nfs
	    100003    3   udp   2049  nfs
	    100003    4   udp   2049  nfs
	    100003    2   tcp   2049  nfs
	    100003    3   tcp   2049  nfs
	    100003    4   tcp   2049  nfs
6. NFS客户端挂载

	显示NFS服务器的共享目录

    	showmount -e 192.168.10.10
	Export list for 192.168.10.10:
	/server   192.168.10.10/24



	Linux挂载NFS的客户端非常简单的命令，先创建挂载目录，然后用 -t nfs 参数挂载就可以了

    	mount -t nfs  192.168.10.10:/server /client


	客户端查看挂载情况
	
    	mount
	192.168.10.10:/server on /client type nfs (rw,addr=192.168.1.5)
或者
    
		df -h
		Filesystem            Size  Used Avail Use% Mounted on
		192.168.1.5:/server  9.7G  1.8G  7.4G  20% /client



	客户端卸载NFS文件命令
	    umount /client



客户机开机自动挂载
客户端可以设置系统启动时自动挂载NFS文件，需要将NFS的共享目录挂载信息写入/etc/fstab/文件，以实现对NFS共享目录的自动挂载。
编辑/etc/fstab文件：

	    vi /etc/fstab
在最后加入如：  

	    192.168.10.10:/server /client nfs defaults 0 0
或者
	
    echo "192.168.10.10:/server /client nfs defaults 0 0" >>/etc/fstab 


然后在客户端简单使用以下命令就可以挂载

    mount /server

## convoy

简介：convoy是Docker 1.8版本引入的卷插件机制。传统的卷管理只能讲本机的目录挂载到容器中，这时数据的备份、共享、迁移都将是一个挑战，基于这些，docker开发了数据卷管理的接口，允许使用第三方插件来管理容器中的数据。 convoy是其中的一中

## 安装

1. 下载软件包

		wget https://github.com/rancher/convoy/releases/download/v0.4.3/convoy.tar.gz
		tar zxvf convoy.tar.gz
		cp convoy/convoy convoy/convoy-pdata_tools /usr/local/bin/
		mkdir -p /etc/docker/plugins/
		bash -c 'echo "unix:///var/run/convoy/convoy.sock" > /etc/docker/plugins/convoy.spec'
2. 测试daemon启动

		convoy daemon --drivers vfs --driver-opts vfs.path=/data/web_data --log=/var/log/convoy/convoy.log --debug=False
3. 修改系统systemd进行后台管理

	1.  vim /usr/lib/systemd/system/convoy.service
	2.  添加：
		
	如下
		
		[Unit]
		
		Description=Convoy server and services
		
		DefaultDependencies=no
		
		After= network.target

		#Requires=convoy.socket
		
		[Service]
		
		Type=simple
		ExecStart=/usr/local/bin/convoy daemon --drivers vfs --driver-opts vfs.path=/data/web_data --log=/var/log/convoy/convoy.log --debug=False >/dev/null 2>&1
		ExecStop=/bin/kill -9 `ps ax | grep "convoy daemon --drivers" | grep -v grep | awk '{ print $1 }'`
		PrivateTmp=true
		
		[Install]
		WantedBy=multi-user.target
4. 详解

[Unit]

Description : 服务的简单描述
Documentation ： 服务文档
After= : 依赖，仅当依赖的服务启动之后再启动自定义的服务单元

[Service]

Type : 启动类型simple、forking、oneshot、notify、dbus

Type=simple（默认值）：systemd认为该服务将立即启动。服务进程不会fork。如果该服务要启动其他服务，不要使用此类型启动，除非该服务是socket激活型。 Type=forking：systemd认为当该服务进程fork，且父进程退出后服务启动成功。对于常规的守护进程（daemon），除非你确定此启动方式无法满足需求，使用此类型启动即可。使用此启动类型应同时指定 PIDFile=，以便systemd能够跟踪服务的主进程。 Type=oneshot：这一选项适用于只执行一项任务、随后立即退出的服务。可能需要同时设置 RemainAfterExit=yes 使得 systemd 在服务进程退出之后仍然认为服务处于激活状态。 Type=notify：与 Type=simple 相同，但约定服务会在就绪后向 systemd 发送一个信号。这一通知的实现由 libsystemd-daemon.so 提供。 Type=dbus：若以此方式启动，当指定的 BusName 出现在DBus系统总线上时，systemd认为服务就绪。 

PIDFile ： pid文件路径 
ExecStartPre ：启动前要做什么，上文中是测试配置文件 －t  
ExecStart：启动 
ExecReload：重载 
ExecStop：停止 
PrivateTmp：True表示给服务分配独立的临时空间

[Install]

WantedBy：服务安装的用户模式，从字面上看，就是想要使用这个服务的有是谁？上文中使用的是：multi-user.target ，就是指想要使用这个服务的目录是多用户



演示地址： [播放连接](https://asciinema.org/a/5okcovyyxtschq46d3kmjgv4c)



[![asciicast](https://asciinema.org/a/5okcovyyxtschq46d3kmjgv4c.png)](https://asciinema.org/a/5okcovyyxtschq46d3kmjgv4c)


## nginx + tomcat 反向代理的实现

实现涉及的技术
	
	1. convoy + NFS 作为共享存储
	2. overlay 作为跨主机共享网络
	3. swarm 作为集群管理工具
	4. docker-compose 作为容器管理工具

实验目标：
	1. 存储可共享
	2. 跨主机网络可共享
	3. nginx 调度两台tomcat实现访问

1. docker-compose.yml

		#========================nginx proxy <start> ============================
		#=======================nginx proxy image link: =========================
		
		nginx_web:
		  image: 192.168.10.99:5000/nginx:1.11
		  restart: always       
		  container_name: nginx_web
		  ports:
		    - "8002:80"
		  volumes:
		    - Nginx:/etc/nginx     # 存储卷
		  volume_driver: convoy    #convoy 存储
		  net: "multihost"      # overlay网络
		
		
		tomcat1:
		  image: 192.168.10.99:5000/tomcat:8.0
		  restart: always
		  container_name: tomcat1
		  expose:
		    - "8080"
		  volumes:
		    - Tomcat:/usr/local/tomcat
		  volume_driver: convoy
		  net: "multihost"
		  environment:
		    - "constraint:node==docker-swarm-server"  #调度分布的主机
		
		tomcat2:
		  image: 192.168.10.99:5000/tomcat:8.0
		  restart: always
		  container_name: tomcat2
		  expose:
		    - "8080"
		  volumes:
		    - Tomcat:/usr/local/tomcat
		  volume_driver: convoy
		  net: "multihost"
		  environment:
		    - "constraint:node==docker-swarm-replica"
		 #========================nginx proxy <end> ===========================
	docker-compose up进行启动
	
	docker-compose ps 查看容器的运行状态

2.  测试跨主机网络连通性

		docker exec -it nginx_web ping tomcat1

		docker exec -it nginx_web ping tomcat2

	测试均无问题

3. 查看convoy 存储

		convoy list
		#看到挂载目录 
			"MountPoint": "/data/web_data/Tomcat"
	查看启动的容器挂载的目录
		
		docker inspect nginx_web | grep -i source #与目录一致

4. 测试tomcat

	修改本地映射的目录中的nginx配置文件后测试

		curl http://192.168.10.99:8002 
	访问正常

	关闭一个tomcat测试，发现有一台不通。
5. 视频演示地址： 播放异常请点击 [视频地址](https://asciinema.org/a/6yj3na1j9g2l9n55qhg28jnxj)

[![asciicast](https://asciinema.org/a/6yj3na1j9g2l9n55qhg28jnxj.png)](https://asciinema.org/a/6yj3na1j9g2l9n55qhg28jnxj)

