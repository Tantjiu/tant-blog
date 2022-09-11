# zookeeper not connected

SpringBoot + dubbo启动时，连接zookeeper出现问题:zookeeper not connected

关于这个问题的一些解决方案

1.检查zookeeper 所在服务器的防火墙是否关闭或对应端口（默认8080，建议修改）是否开放
2.检查zookeeper 所在服务器的ip和yml配置中的ip是否对应
3.检查zookeeper 是否成功启动。
4.在yml配置文件中增加新的配置，提高连接zookeeper 的访问超时时间(有可能是虚拟机网络不稳定造成连接zk的时候，出现超时)

dubbo中的默认超时配置是3秒。

~~~yml
dubbo:
	config-center:
		timeout: 10000
~~~

5.在启动类型上增加新的注解， @EnableDubboConfig。 人为强制要求dubbo-spring-boot-starter扫描配置并加载。 dubbo是自动扫描配置并加载的。
6.修改版本。降低spring-boot和dubbo-spring-boot-starter版本。（版本问题是win10操作系统对权限管理加强后，导致的结果。）
	6.1 先降低dubbo-spring-boot-starter 到 2.7.3 -> 2.7.0
	6.2 再考虑降低spring-boot版本 到 2.2.0 -> 2.1.10 -> 2.0.2

**同时虚拟机的网络也是有一定的影响，如何提高虚拟机的网络稳定呢？**　　

1. 关闭不必要的网卡
2. 关闭所有的热点软件
3. 关闭windows防火墙