# docker 笔记
1. 公司测试环境已经部署docker，并实现高可用，能对所有容器、镜像管理，有私有仓库，k/v存储采用etcd。
2. docker-compose yml语法学习，用yml来组建容器间的关系（links、架构），并实现跨主机自动创建容器、自动根据yml文件来建立关系
3. 用yml组件nginx-web1-web2架构，实现负载均衡，反向代理，但遇到二层网络不通的问题。
4. docker 1.9 用overlay网络可跨主机实现容器间通信，但1.9在构建容器网络架构时遇到该版本的bug：--net=<network>,vxlan无法创建的问题，google搜索升级系统内核到3.10.237以上版本即可。升级后从新编写compose yml，发现links需要高版本docker引擎支持。全部升级完成后。创建overlay网络，跨主机创建容器，发现不同主机间共同二层网络不通。但在网络里面能看到所有容器的ip地址，该问题暂时没有解决。

5. 遇到的坑：

	1. compose  : links  需要版本version 2 ，但是v2 需要1.10以上的docker engine
	2. compose 调度问题：采用overlay网络（container跨主机网络通信），当手工调度时报错error creating vxlan interface: file exists 此问题为docker 1.9的bug，与内核3.10不兼容导致。
	3. 升级到1.11版本后，解决了compose中容器创建依赖net问题，但虚拟二层网络不通。

6. compose yaml语法讲解：


>     nginx:
>       image: jwilder/nginx-proxy #镜像名称
>       restart: always            #自启动（例如宿主机重启）
>       container_name: nginx_web  #容器名称
>       ports:
>         - "8001:80"              #映射端口
>       volumes:
>         - /var/run/docker.sock:/tmp/docker.sock:ro #只读映射挂载
>       net: "multihost"           #指定网络名称
>       environment:
>         - "constraint:node==docker-swarm-replica"  #分配到节点
>         - "VIRTUAL_HOST=web2,web3"  #nginx 虚拟主机配置
>         - "VIRTUAL_HOST=web3"
>     
>     web:
>       image: 192.168.10.99:5000/centos-lamp
>       restart: always
>       container_name: web
>       volumes:
>         - /data/centos/html:/var/www/html
>       net: "multihost"
>       environment:
>         - "constraint:node==docker-swarm-server"
>     
>     web2:
>       image: 192.168.10.99:5000/centos-lamp
>       restart: always
>       container_name: web2
>       volumes:
>         - /data/centos2/html:/var/www/html
>       net: "multihost"
>       environment:
>         - "constraint:node==docker-swarm-replica"
>     
>     
>     web3:
>       image: 192.168.10.99:5000/centos-lamp
>       restart: always
>       container_name: web3
>       volumes:
>         - /data/centos3/html:/var/www/html
>       net: "multihost"
>       environment
>       以上yml文件创建四台container，并且之间建立关系，组成体系的网络服务架构


  * 用docker-compose build；docker-compose up 来创建。
  * docker-compose ps 来查看

=======================6.15问题解决=======================

1. 解决虚拟2层网络不通问题
  
	1. 升级系统内核版本为3.10.327，升级docker engine 1.22.2
	2. 修改/usr/lib/systemd/system/docker.service
	
	`ExecStart=/usr/bin/docker daemon -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --insecure-registry 192.168.10.99:5000 --cluster-store=etcd://192.168.10.99:2379 --cluster-advertise=enp3s0:2375 --storage-driver=devicemapper  --selinux-enabled`
    解决2层网络
2. 解决监控问题

	1. 编写监控采集程序对主机所有容器进行cpu/mem/network数据采集
	2. 编写发现程序对所有主机上的container进行自动发现
	3. 采用zabbix前端和graph功能对接后台程序实现自动发现并监控

3. docker-compose优化

	1. compose实现对所有容器跨主机管理，以及管理容器业务架构(eg:nginx-web-web2)
	2. compose结合dockerfile，自动完成容器构建、创建
	3. compose自动删除重复创建的容器

4. 访问的问题

	1. 容器ip随时可变，采用端口映射方式访问业务
	2. 绑定域名后，访问时加端口极不方便，可采用：

		`ip:hostip:containerIP` #可以为主机绑定虚拟ip

		`192.168.10.222:80:80` #用虚拟ip的80端口绑定虚拟机的80端口

遇到的新问题：

1. 数据卷容器无法跨主机共享，--volumes-from 与 --net无法跨越主机
2. 采用NFS后用volume，但管理、性能都较差，较好的方案是用社区提供的convoy工具结合NFS可以实现跨主机存储和共享
3. 第三方插件flocker 特性：
	1. 部署一个多容器应用到多个主机
	2. 在不同主机之间移动容器以及对应的卷
	3. 当容器更改主机时对数据卷进行绑定和解绑
	4. 在不同的服务器之间移植本地数据卷
    
	在1.2.0版本是不支持卷共享，逻辑架构比较复杂，flocker control service 是一个单节点部署服务，容灾能力较差。

总结：采用NFS + convoy实现跨主机存储和共享的方案 （待测试）

问题：
	
	1. 开发人员如何登录容器进行开发工作，若通过ssh，则需要固定ip
	2. 有固定ip后则需要进行端口映射访问，即每个开发环境的容器至少需要映射3个端口，当容器多了以后，很难进行管理
	

=============================6.22 问题及解决========================

1. 启动convoy daemon的问题

	启动总是在前台进行，需要采用后台管理进程方式启动，故编写脚本/etc/init.d/convoy
	但是发现redhat/centos 7版本后都是靠systemd进行管理，所以在/etc/init.d/下编写脚本进行启动是不可用的（可能需要该多个地方，暂时没发现）：解决方式：
	
		在/usr/lib/systemd/system/编写convoy.service
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
		即可解决
2. convoy 删除卷的问题

	convoy crate/delete 都是调用convoy的api来进行创建/删除卷的，但是删除后日志每分钟都会报：

		{"level":"debug","msg":"Response:  {\n\t\"Err\": \"Couldn't find volume.\"\n}","pkg":"daemon","time":"2016-06-22T17:58:06+08:00"}
		{"level":"error","msg":"Handler not found: POST /VolumeDriver.List","pkg":"daemon","time":"2016-06-22T17:58:14+08:00"}
		{"level":"debug","msg":"Handle plugin volume path: POST /VolumeDriver.Path","pkg":"daemon","time":"2016-06-22T17:58:14+08:00"}
		{"level":"debug","msg":"Request from docker: \u0026{Tomcat-nginx map[]}","pkg":"daemon","time":"2016-06-22T17:58:14+08:00"}
		{"level":"debug","msg":"Response:  {\n\t\"Err\": \"Couldn't find volume.\"\n}","pkg":"daemon","time":"2016-06-22T17:58:14+08:00"}
		{"level":"error","msg":"Handler not found: POST /VolumeDriver.List","pkg":"daemon","time":"2016-06-22T17:58:37+08:00"}
		{"level":"debug","msg":"Handle plugin volume path: POST /VolumeDriver.Path","pkg":"daemon","time":"2016-06-22T17:58:37+08:00"}
		{"level":"debug","msg":"Request from docker: \u0026{Tomcat-nginx map[]}","pkg":"daemon","time":"2016-06-22T17:58:37+08:00"}
		{"level":"debug","msg":"Response:  {\n\t\"Err\": \"Couldn't find volume.\"\n}","pkg":"daemon","time":"2016-06-22T17:58:37+08:00"}
	而且：在启动daemon时可以使用 --debug=false来关闭debug，但是不生效。可能是convoy的bug问题。

3. 使用NFS结合convoy的问题

	在NFS客户端的共享目录下创建文件/目录都是以nfsnobody 所以造成在创建容器时
		
		docker run -id -d -p 8881:8080 -v Tomcat:/usr/local/tomcat --name cjy_tomcat --volume-driver=convoy 192.168.10.99:5000/tomcat:8.0
	总提示权限拒绝，或是无法找到volume，或是volume名称唯一的一些奇怪问题。
	
	解决办法:
		
		1.创建用户及挂卷需要root权限，所以在NFS服务端更改：
			 /data/web_data 192.168.10.0/24(rw,sync,no_root_squash,anonuid=0,anongid=0)
		2. 无法找到volume：
			需要在所有convoy机器上，convoy creat 创建一样的卷，才能找到并识别。
			例如在node1节点新建一个卷，并想在node2上的容器通过convoy共享，需要在node2上创建相同名称的卷，并不冲突。
		3. volume name must be unique 问题：
			之前创建容器时指定了名称一样的卷，在/var/lib/docker/volumes可以看到卷名，但是没有挂载到convoy上，所以出现了该问题，
			解决方式：删除上述路径中的卷（保证没有容器使用），然后在进行创建就没有问题了。
4. compose无法使用convoy的问题

	在docker-compose.yaml文件中volume_driver指定convoy但是创建的容器没有挂载指定卷
	解决方式：
		
		convoy卷挂载不能为空，需要手工创建一个容器，并进行挂载后在用compose 是没问题的。
	
=========================6.24 问题==================================

1. docker重启对convoy的影响 

	重启docker后所有挂载convoy卷的容器都无法启动，显示Volume.Get无法找到。目前只有进行重建container来解决。此问题必须修复 