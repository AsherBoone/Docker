#	docker-registry install
## 一、搭建私有仓库的优点
* 可独立开发和运维私有仓库
* 可节省贷款资源
* 有独立的账户管理系统
* 增加了定制化功能

## 二、搭建私有仓库
### 前提
	搭建私有仓库的前提是部署Docker Private Registry，有官方提供的镜像进行安装。
	
### 安装
1. 下载官方提供的registray镜像

	`docker pull registry`  # 默认镜像是ubunt,registry-1.2.2版本
2. 安装仓库
	
	`docker run -d -name local-registry --restart=always -p 5000:5000 -v /docker/images:/tmp/registry registry`

	* -d: 以后台模式运行
	* --name: 指定创建容器名称
	* --restart=always：使容器一直保持运行(主机重启后也能随之启动)
	* -p: 指定本地端口与容器端口的映射情况
	* -v：指定本地与容器路径的映射，/tmp/registry是容器中存放镜像的位置，需要映射到本地进行备份保存。
	
3. 镜像的上传

	`docker tag centos:latest 192.168.30.8:5000/centos`
	`docker push 192.168.30.8:5000/centos`
	
4. 查看仓库镜像

	`docker search 192.168.30.8:5000/library`
	
	`docker search 192.168.30.8:5000/centos`
	 
	`curl http://192.168.30.8:5000/search`
5. 下载本地镜像
	`docker pull 192.168.30.8:5000/centos:latest`
	
	`docker pull 192.168.30.8:5000/ubuntu:14.04`

## 构建反向代理
### 说明
	在实际使用中，暴露主机的端口的方法是不安全的，如果registry没有配置访问代理，任何用户都是可以直接通过端口访问，因此，设计时需要为其加上https反向代理。
后期会根据需要进行添加
***
## Index及仓库高级功能
### 说明
	1.安装上面的搭建方式只具备镜像存储和访问功能，而不具备账户管理功能，要实现完整的解决方案，需要设计DockerIndex用户管理系统。
	2.仓库镜像高级查询和记录仓库镜像活动（具备时间通知功能）
后期根据需要在做研究
***
