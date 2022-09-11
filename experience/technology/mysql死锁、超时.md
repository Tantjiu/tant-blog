# Mysql使用kill命令解决死锁问题(杀死某条正在执行的sql语句)

在使用mysql运行某些语句时，会因数据量太大而导致死锁，这个时候，就需要kill掉某个语句，KILL命令的语法格式如下：

~~~
KILL [CONNECTION | QUERY] thread_id
~~~

查看哪些线程正在运行

~~~
SHOW PROCESSLIST
~~~

终止一个线程

~~~
KILL 56655
~~~

注意：kill全部线程需要有管理员权限，否则只能kill自己创建的