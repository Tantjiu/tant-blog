# Bean with name 'xxxxxx' has been injected into other beans

依赖循环：现在有一个ServiceA需要调用ServiceB的方法，那么ServiceA就依赖于ServiceB，那在ServiceB中再调用ServiceA的方法，就形成了循环依赖。Spring在初始化bean的时候就不知道先初始化哪个bean就会报错。

解决办法：

1、在其中一个service的bean上加@Lazy 

2、不要互相调用