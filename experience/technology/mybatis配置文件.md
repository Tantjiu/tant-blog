# **Invalid** **bound** **statement** not found

两种原因：

## 1.mapper写在了SRC下

这样配置的话即使你在properties里面配置：

~~~properties
mybatis.mapper-locations= classpath:com/demo/mapper/.xml
~~~

也是没有用的，因为编译的时候这个xml文件并没有被自动拉到target里面

编译的时候读取的是.java文件而不是.xml，

想要编译时读取.xml，pom文件里面加上：

~~~xml
<build>
		<resources>
            <resource>
                <directory>src/main/java</directory>
                <includes>
                    <include>**/*.xml</include>
                </includes>
            </resource>
            <resource>
                <directory>src/main/resources</directory>
            </resource>
        </resources>
</build>
~~~

## 2.xml错误

检查namespace、id和mapper里面的方法名、parameterType