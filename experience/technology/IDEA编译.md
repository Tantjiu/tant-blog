# IDEA编译提示：程序包xxx不存在，找不到符号

在开发时，增加新的模块或者导入新的项目后提示”xxx程序包不存在“问题

- #### 检查项目的maven依赖没有问题

  查看项目的Dependencies中是否有提示报错不存在的依赖包或者该依赖包是否报错，如果不存在或者报错，则检查pom文件中的依赖是否填写正确或者本地仓储是否存在

- #### 检查本地仓库是否存在依赖的jar包

  点击设置“File --> Settings --> Build, Execution, Deployment --> Build Tools --> Maven ”中“Local repository”这一项对应的repository目录下面是否存在依赖的maven依赖包，没有则需要检查maven仓库路径是否正确

- #### 检查maven更新依赖

  点击右侧的maven工具栏中“Reimport All Maven Project”可以重新导入maven依赖或者右键点击当前提示报错的项目“Maven --> Reimport”

- #### 检查maven单独编译是否通过

  执行`mvn -X -DskipTests=true compile`，如果能正常执行则代表maven本身依赖和编译没有问题而是idea的问题，如果不行则应该根据提示信息检查（如：maven配置，依赖jar包是否存在等）

- #### idea工具缓存问题

  “File --> Invalidate Caches” 选中“INVALIDATE AND RESTART”，自动重启idea工具

- #### idea配置文件问题

  删除工程目录下面的“.idea”文件夹，重新启动idea工具