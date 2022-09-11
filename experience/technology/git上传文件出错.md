# git上传文件出错[rejected] master -> master (fetch first) error: failed to push some refs to '

用git上传文件的时候遇到的一些问题

## 一、远程仓库已有文件，而本地仓库没有

出错：! [rejected] master -> master (fetch first) error: failed to push some refs to ' 。。。'

出现这个问题是因为github中的README.md文件不在本地代码目录中，可以通过如下命令进行代码合并

~~~git
git pull --rebase origin master
~~~

## 二、master是受保护

推送代码时报错：! [remote rejected] master -> master (pre-receive hook declined)

一般公司新人会遇到，推代码不要直接推master，一定要先新建自己的分支，再把master的代码合并到自己分支上解决冲突问题，没问题后让你的领导去合回到master
