# Access denied for user ''@'localhost' (using password: NO)

idea启动sql连接远程数据库时发生错误：

sql连接配置问题：

~~~yml
# 我写的配置文件，用户名和密码前面加data是IDEA提示的
spring:
  datasource:
    data-username: root
    data-password: root
    url: jdbc:mysql://localhost:3307/jdbc?useSSL=false
    driver-class-name: com.mysql.jdbc.Driver
~~~

正确的配置文件

~~~yml
spring:
  datasource:
    username: root
    password: 123456
    url: jdbc:mysql://locaohost:3307/jdbc?useSSL=false
    driver-class-name: com.mysql.jdbc.Driver
~~~

