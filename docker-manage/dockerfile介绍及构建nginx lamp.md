# dockerfile构建nginx、tomcat、lamp

介绍dockerfile的使用以及如何利用dockerfile构建自定义镜像

## dockerfile讲解

dockerfile中有自己的语法和关键字，下面开始讲解：

1. FEOM：base 镜像，表示以哪个镜像为基础镜像。
2. MAINTAINER：维护者的信息，表示是有谁管理的。
3. RUN：需要做什么，会在一个新的容器中执行任何命令，然后把执行后的改变提交到当前镜像，提交后的镜像会被用于dockerfile中定义的下一步操作
	1. `RUN <command>`     #将会调用/bin/sh -c <command> ；RUN yum -y install ssh
	2. `RUN ["executable", "param1", "param2"]` #将会调用exec执行 以避免有些时候shell方式执行时的传递参数问题
4. ADD：复制文件的指令，
	`ADD <src> <dest>`  #从指定路径拷贝一个文件/目录到容器指定的路径中
5. CMD：提供了容器默认的执行命令。Dockerfile只允许CMD命令使用一次。使用多个最后一个生效。
	`CMD ["executable", "param1", "param2"]`;eg:`CMD ["/usr/sbin/sshd", "-D"]`
6. EXPOSE：该指令用来告诉docker这个容器在运行时会监听哪些端口
	`EXPOSE 22 443`
7. COPY：COPY <src> <dest> ;用法与ADD相同，不过<src>不支持url，所以在使用`docker build - < somefile`是该指令不能使用。
8. ENTRYPOINT：配置容器一个可执行的命令，这意味着在每次使用镜像创建容器时一个特定的应用程序可以被设置为默认程序。同时也意味着该镜像每次被调用时仅能运行指定的应用。和CMD的区别是：
	
	1. CMD ["echo"] ; `docker run XXXXX echo foo`;打印结果是：foo ；被替换掉了
	2. ENTRYPOINT ["echo"];`docker run xx echo foo`;打印：echo foo
	3. 另外，在Dockerfile中，ENTRYPOINT指定的参数比运行docker run时指定的参数更靠前，比如：
		ENTRYPOINT ["echo","foo"]；执行`docker run XX bar`;相当于执行了echo foo bar； 结果为：foo bar   
	4. 只能存在一个ENTRYPOINT。
8. WORKDIR：用于设置dockerfile中的RUN、CMD、ENTRYPOINT指令执行命令的工作目录，可以多次切换(相当于cd命令)
	`WORKDIR /path/to/workdir`
9. VOLUME：该指令用来设置一个挂载点，可以用来让其他容器挂载实现数据共享或对容器数据的备份、恢复或迁移。
	`VOLUME ['/data']`
10. ENV：该指令用于设置环境变量，在Dockerfile中这些设置的环境变量也会影响到RUN指令，当运行生成的镜像时这些环境变量依然有效，如果需要在运行时更改这些环境变量可以在运行docker run时添加–env <key>=<value>参数来修改。
	`ENV <key> <value>`
	ENV LANG en_US.UTF-8
	ENV LC_ALL en_US.UTF-8
11. USER：镜像正在运行时设置一个UID。语法：
	`USER <uid>`
	ENTRYPOINT ["memcached"]
	USER daemon
12. ONBUILD：ONBUILD 指定的命令在构建镜像时并不执行，而是在它的子镜像中执行
	格式为 ONBUILD [INSTRUCTION]，配置当所创建的镜像作为其它新创建镜像的基础镜像时，所执行的操作指令。
	ONBUILD ADD . /app/src
	ONBUILD RUN /usr/local/bin/python-build --dir /app/src
	<http://edu.cnzz.cn/201509/96984dd8.shtml>
	
## dockerfile 构建nginx实例

	#NGINX 1.10
	FROM centos:latest
	MAINTAINER chenjiayun <jiayun@gyyx.cn>
	#COPY ./nginx.repo /etc/yum.repos.d/nginx.repo
	RUN rpm -ivh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
	# Install Nginx
	RUN /usr/bin/yum install -y nginx \
	&& /usr/bin/yum clean all && rm -rf /etc/nginx/conf.d/* /tmp/* /var/tmp/* /var/run/nginx \
	 && mkdir /var/run/nginx && chown nginx /var/run/nginx && chmod 777 /var/run/nginx \
	# rm -f /etc/nginx/nginx.conf
	# forward request and error logs to docker log collector
	&& ln -sf /dev/stdout /var/log/nginx/docker-access.log \
	&& ln -sf /dev/stderr /var/log/nginx/docker-error.log
	
	# nginx site conf
	
	#COPY ./nginx.conf /etc/nginx/nginx.conf
	
	EXPOSE 80 443
	
	CMD ["nginx","-g","daemon off;"]


<https://github.com/nanderson94/dockerfile-nginx>

然后用docker build -t centos/nginx1.10 .  #进行构建

	[#37#root@docker-server ~/df_test]#docker build -t nginx_003 .
	Sending build context to Docker daemon  2.56 kB
	Step 1 : FROM centos:latest
	 ---> eeb3a076a0be
	Step 2 : MAINTAINER chenjiayun <jiayun@gyyx.cn>
	 ---> Using cache
	 ---> 58cf9f610303
	Step 3 : RUN rpm -ivh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
	 ---> Using cache
	 ---> 3539d142a4d1
	Step 4 : RUN /usr/bin/yum install -y nginx && /usr/bin/yum clean all && rm -rf /etc/nginx/conf.d/* /tmp/* /var/tmp/* /var/run/nginx  && mkdir /var/run/nginx && chown nginx /var/run/nginx && chmod 777 /var/run/nginx && ln -sf /dev/stdout /var/log/nginx/docker-access.log && ln -sf /dev/stderr /var/log/nginx/docker-error.log
	 ---> Using cache
	 ---> afd8d9004e98
	Step 5 : VOLUME /test/ /AAB
	 ---> Running in 28be3dd25c2a
	 ---> 0f6584a9d561
	Removing intermediate container 28be3dd25c2a
	Step 6 : EXPOSE 80 443
	 ---> Running in 5f02c169776a
	 ---> bf858a3080cc
	Removing intermediate container 5f02c169776a
	Step 7 : CMD nginx -g daemon off;
	 ---> Running in 2d08ec1a3903
	 ---> 352dec2d0e0d
	Removing intermediate container 2d08ec1a3903
	Successfully built 352dec2d0e0d

docker run -it -d --name cjy_test -p 8080:80 -p 4433:443 centos/nginx1.10 启动 



## tomcat、lamp构建

同nginx构建方式。主要注意关键字的用法。eg：RUN 进行安装、配置；VOLUE：可以挂载多个目录
eg:

	VOLUME ["/etc/nginx/sites-enabled", "/etc/nginx/certs", "/etc/nginx/conf.d", "/var/log/nginx", "/var/www/html"]

tomcat/lamp搭建方式同nginx，一般我们都从官方下载tomcat、lamp镜像，然后在封装自己需要的软件在build成镜像放置我们的私有仓库中。若有特殊需求，可用base镜像来构建自定义的image。