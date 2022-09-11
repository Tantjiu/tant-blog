# mysql登录报错：mysql: error while loading shared libraries: libncurses.so.5: cannot open shared object fi...

系统是centos8，安装完mysql之后，mysql命令登录不成功

报错：mysql: error while loading shared libraries: libncurses.so.5: cannot open shared object file: No such file or directory。

猜测应该和系统版本有关，运行：

~~~linux
yum install libncurses*
~~~

