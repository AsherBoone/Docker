# docker下Nginx+Tomcat

简介： 用docker容器实现Tomcat负载均衡、反向代理，HA；

## docker 下nginx+tomcat架构1

![nginx](http://gogs.gyyx.cn/chenjiayun/Docker/raw/master/picture/nginx+tomcat01.png)

注释： 存储用NFS + Convoy，可以跨主机存储共享；网络用overlay跨主机网络共享；监控用zabbix + 自编程序代码实现自动迁移功能(二期开发)。

1. 管理：该架构采用docker-compose文件来管理所有主机容器。
2. 分布：tomcat分布在不同主机上，采用共享存储，可以提供相同/不同的业务。
3. 存储：用nginx对tomcat进行代理，nginx配置文件以及tomcat数据文件均在nfs上共享，修改nfs上的数据即可完成nginx配置和tomcat网站数据更新。
4. 公网访问：使用绑定公网ip到host然后用该ip绑定nginx容器即可完成公网的访问。
5. 迁移：从ip/port/host/container/access多维度检测容器的可用性，若自动重启容器有异常立即解绑该host上公网ip，删除容器并在新主机上创建新nginx容器并绑定公网ip。

问题：
1. Nginx没有高可用，自动迁移会有几秒的中断，compose不是很完善，若触发新bug会有1-2分钟中断
2. 当访问量高Nginx会有瓶颈，也存在单点故障。


## docker 下nginx+tomcat架构2


![nginx](http://gogs.gyyx.cn/chenjiayun/Docker/raw/master/picture/nginx+tomcat.png)

1. 该架构增加keepalived实现高可用方案，解决方案一中的1、2问题。
2. 用docker-compose文件固定keepalived容器所在的host，避免主机宕机影响正常访问。

方案二问题：

1. 容器keepalived绑定公网地址是否可以通过docker0在由iptables规则连接外网？
2. 如果keepalived的vip是一个容器内部的ip地址，通过host映射与本地网络连通，是否可以绑定公网域名映射访问？
