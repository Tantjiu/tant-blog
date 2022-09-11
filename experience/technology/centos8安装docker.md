# centos8安装docker

首先安装 安装 yum-utils 为了使用 yum-config-manager 命令

~~~
yum install yum-utils
~~~

设置镜像仓库地址为阿里云镜像

~~~
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
~~~

安装依赖 containerd.io，不安装会报错

~~~
dnf install https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/stable/Packages/containerd.io-1.2.13-3.1.el7.x86_64.rpm
~~~

安装 docker-ce

~~~
dnf install docker-ce
~~~

设置开机启动、启动docker

~~~
systemctl enable docker.service
systemctl start docker.service
~~~

然后去阿里云上搜镜像服务 -> 镜像加速器

按提示设置，完成