## IDEA报错：ERROR:JAVA: 错误: 不支持发行版本 5 解决方法

**出现原因：本地配置jdk和idea默认的jdk不匹配**

## 方法一：修改IDEA设置

项目、模块、编译的Java版本都修改一下即可

## 方法二：修改pom.xml配置文件

~~~xml
<properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
</properties>
~~~

## 方法三：修改maven的setting文件

~~~xml
<profile>
     <id>jdk-1.8</id>
     <activation>
         <activeByDefault>true</activeByDefault>
         <jdk>8</jdk>
     </activation>
     <properties>
         <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
         <maven.compiler.source>8</maven.compiler.source>
         <maven.compiler.target>8</maven.compiler.target> 
     </properties> 
</profile>
~~~

