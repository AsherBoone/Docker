# zabbix docker monitor

背景：目前使用docker官方提供的swarm作为docker集群管理工具，但swarm本身没有集成监控系统，所以得考虑开发一套监控或是采用第三方的监控工具来实现。目前遇到的问题有：
	
	1. 如果采用其他公司的监控平台，也就是需要服务器开放端口，有很大风险。
	2. 采用开源工具，目前有很多工具，但不利于操作和扩展，且需要较长时间学习

所以，我会基于以下标准来评估docker监控工具：

	1. 易于部署
	2. 监控呈现的详细度（graph、screen）
	3. 监控报警能力
	4. docker资源（cpu、mem、network）、非docker资源（宿主机、docker数量、状态）
	5. 扩展能力 （插件编写）
	6. 成本

总结：后端编写代码调用docker api来采集指定监控数据，前端交给zabbix做数据展示及告警处理。扩展方便，可以依靠监控来实现自动创建已发生异常的容器。

# zabbix监控docker的实例

1. 自动发现host上的容器，主动监控,并有丰富的告警策略
![docker](http://gogs.gyyx.cn/chenjiayun/Docker/raw/master/picture/zabbix-docker01.png)

2. 实时数据展示（60s,30s)
![docker](http://gogs.gyyx.cn/chenjiayun/Docker/raw/master/picture/zabbix-docker02.png)

3. 单一容器监控数据绘图
![docker](http://gogs.gyyx.cn/chenjiayun/Docker/raw/master/picture/zabbix-docker03.png)

4. 多容器监控数据绘图
![docker](http://gogs.gyyx.cn/chenjiayun/Docker/raw/master/picture/zabbix-docker04.png)

5. 根据告警执行远程脚本，可自动恢复或自动创建容器
![docker](http://gogs.gyyx.cn/chenjiayun/Docker/raw/master/picture/zabbix-docker05.png) 

# 扩展功能

容器异常自动删除并创建，后期开发 ，周期预计3天
