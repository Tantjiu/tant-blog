# mybatis拦截器

## 一. 介绍

- mybatis拦截器设计的初衷是为了供用户在某些时候可以实现自己的逻辑,而不必去动mybatis固有的逻辑.通过拦截器可以拦截某些方法的调用,可以选择在这些被拦截的方法前后加上某些逻辑,也可以执行这些被拦截的方法时执行自己的逻辑而不是被拦截的方法

### 1.1 核心对象

#### 1.1.0 核心对象介绍

| Mybatis核心对象  | 解释                                                         |
| ---------------- | ------------------------------------------------------------ |
| SqlSession       | 作为MyBatis工作的主要顶层API，表示和数据库交互的会话，完成必要数据库增删改查功能 |
| Executor         | MyBatis执行器，是MyBatis 调度的核心，负责SQL语句的生成和查询缓存的维护 |
| StatementHandler | 封装了JDBC Statement操作，负责对JDBC statement 的操作，如设置参数、将Statement结果集转换成List集合 |
| ParameterHandler | 负责对用户传递的参数转换成JDBC Statement 所需要的参数        |
| ResultSetHandler | 负责将JDBC返回的ResultSet结果集对象转换成List类型的集合；    |
| TypeHandler      | 负责java数据类型和jdbc数据类型之间的映射和转换               |
| MappedStatement  | MappedStatement维护了一条mapper.xml文件里面 select 、update、delete、insert节点的封装 |
| SqlSource        | 负责根据用户传递的parameterObject，动态地生成SQL语句，将信息封装到BoundSql对象中，并返回 |
| BoundSql         | 表示动态生成的SQL语句以及相应的参数信息,可以执行sql的封装    |
| Configuration    | MyBatis所有的配置信息都维持在Configuration对象之中           |

#### 1.1.1 SqlSource

SqlSource为SQL 来源接口。它从 Mapper [XML](https://so.csdn.net/so/search?q=XML&spm=1001.2101.3001.7020) 或方法注解上，生成一条 SQL

具有多个实现类,区别为:

- **DynamicSqlSource** 表示带有`${}`占位符的sql语句，如：`SELECT * FROM ${tableName} where  ID = ?`
  - 动态的 SqlSource 实现类

- **RawSqlSource** 表示可能带有`#{}`占位符的sql，如：`select * from Blog where id = #{id}`
  - 如果当前的sql语句不是动态sql，也就是说sql语句中不包含`${}`占位符的话，RawSqlSource会将当前sql中的`#{}`占位符替换为`?`，并维护其参数映射关系，最终返回一个StaticSqlSource对象。

- **StaticSqlSource** 表示不带有占位符且可能会包含 `?` 的sql语句，即持有不包含 `#{}` 和 `${}` 占位符的sql,如：`select * from post order by id`

- **ProviderSqlSource** 表示sql来自基于方法上的 `@ProviderXXX` 注解所定义的sql

> **RawSqlSource的执行时机为在mybatis初始化时完成sql的解析，而DynamicSqlSource的执行时机为实际执行sql语句之前**

#### 1.1.2 SqkSource解析

SqlSource的入口在`XMLScriptBuilder#parseScriptNode`，如果解析后的SqlNode是动态节点，则创建DynamicSqlSource的实例，如果SqlNode是静态节点则创建RawSqlSource的实例,代码如下

~~~Java
public SqlSource parseScriptNode() {
  //解析动态标签  
  MixedSqlNode rootSqlNode = parseDynamicTags(context);
  SqlSource sqlSource;
  if (isDynamic) {//如果是动态sql，则创建DynamicSqlSource实例
    sqlSource = new DynamicSqlSource(configuration, rootSqlNode);
  } else {
    sqlSource = new RawSqlSource(configuration, rootSqlNode, parameterType);
  }
  return sqlSource;
}
~~~

#### 1.1.3 SqlSourceBuilder

继承 BaseBuilder 抽象类，SqlSource 构建器，负责将 SQL 语句中的 `#{}` 替换成相应的 `?` 占位符，并获取该 `?` 占位符对应的 `org.apache.ibatis.mapping.ParameterMapping` 对象

#### 1.1.4 parse方法

parse方法为SqlSourceBuilder类的核心方法，该方法可以将sql中的`#{}`占位符替换为`?`

### 1.2 拦截对象

mybatis拦截器并不是每个对象里的方法都会拦截,只会拦截以下四个拦截器

- Executor
  - 所有mapper语句的执行都是通过Executor进行的,该接口是mybatis的核心接口.从其定义的方法来看,对应的增删改语句是通过Executor接口的update方法进行的，查询是通过query方法进行的。Executor里面常用拦截方法如下所示

~~~Java
public interface Executor {
	...
    /**
     * 执行update/insert/delete
     */
    int update(MappedStatement ms, Object parameter) throws SQLException;

    /**
     * 执行查询,先在缓存里面查找
     */
    <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey cacheKey, BoundSql boundSql) throws SQLException;

    /**
     * 执行查询
     */
    <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException;

    /**
     * 执行查询，查询结果放在Cursor里面
     */
    <E> Cursor<E> queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds) throws SQLException;
	...
}
~~~

- ParameterHandler
  - 用来设置参数规则,当StatementHandler使用prepare()方法后,接下来就是用该接口设置参数,如果又对参数做自定义逻辑处理的话,可以通过拦截ParameterHandler来实现,方法参数如下

~~~Java
public interface ParameterHandler {

 ...

 /**
  * 设置参数规则的时候调用 -- PreparedStatement
  */
 void setParameters(PreparedStatement ps) throws SQLException;
 ...
}
~~~

- StatementHandler

StatementHandler负责处理mybatis与JDBC之间statement的交互

> 一般只拦截StatementHandler中的prepare()方法

~~~Java
public interface StatementHandler {

    ...

    /**
     * 从连接中获取一个Statement
     */
    Statement prepare(Connection connection, Integer transactionTimeout)
            throws SQLException;

    /**
     * 设置statement执行里所需的参数
     */
    void parameterize(Statement statement)
            throws SQLException;

    /**
     * 批量
     */
    void batch(Statement statement)
            throws SQLException;

    /**
     * 更新：update/insert/delete语句
     */
    int update(Statement statement)
            throws SQLException;

    /**
     * 执行查询
     */
    <E> List<E> query(Statement statement, ResultHandler resultHandler)
            throws SQLException;

    <E> Cursor<E> queryCursor(Statement statement)
            throws SQLException;

    ...

}
~~~

- 在Mybatis里面RoutingStatementHandler是SimpleStatementHandler(对应Statement)、PreparedStatementHandler(对应PreparedStatement)、CallableStatementHandler(对应CallableStatement)的路由类，所有需要拦截StatementHandler里面的方法的时候，对RoutingStatementHandler做拦截处理就可以了，如下的写法可以过滤掉一些不必要的拦截类。

~~~Java
@Intercepts({
        @Signature(
                type = StatementHandler.class,
                method = "prepare",
                args = {Connection.class, Integer.class}
        )
})
public class TableShardInterceptor implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        if (invocation.getTarget() instanceof RoutingStatementHandler) {
            // TODO: 做自己的逻辑
        }
        return invocation.proceed();
    }

    @Override
    public Object plugin(Object target) {
        // 当目标类是StatementHandler类型时，才包装目标类，否者直接返回目标本身,减少目标被代理的次数
        return (target instanceof RoutingStatementHandler) ? Plugin.wrap(target, this) : target;
    }

    @Override
    public void setProperties(Properties properties) {

    }
}
~~~

- ResultSetHandler
  - 对查询的结果做处理,所以如果你有需求需要对返回结果做特殊处理的情况,可以去拦截ResultSetHandler的处理.常用方法如下

~~~Java
public interface ResultSetHandler {

    /**
     * 将Statement执行后产生的结果集（可能有多个结果集）映射为结果列表
     */
    <E> List<E> handleResultSets(Statement stmt) throws SQLException;
    <E> Cursor<E> handleCursorResultSets(Statement stmt) throws SQLException;

    /**
     * 处理存储过程执行后的输出参数
     */
    void handleOutputParameters(CallableStatement cs) throws SQLException;

}
~~~

#### 1.2.1 拦截顺序

- 拦截的执行顺序是Executor->StatementHandler->ParameterHandler->ResultHandler

## 二. 使用

### 2.1 自定义拦截器类

自定义拦截器类需要实现mybatis提供的Interceptor接口,并需要在自定义拦截器类上加上@Interceptor注解

#### 2.1.1 Interceptor接口

接口中的方法如下

~~~java
public interface Interceptor {

    /**
     * 代理对象每次调用的方法，就是要进行拦截的时候要执行的方法。在这个方法里面做我们自定义的逻辑处理
     */
    Object intercept(Invocation invocation) throws Throwable;

    /**
     * plugin方法是拦截器用于封装目标对象的，通过该方法我们可以返回目标对象本身，也可以返回一个它的代理
     *
     * 当返回的是代理的时候我们可以对其中的方法进行拦截来调用intercept方法 -- Plugin.wrap(target, this)
     * 当返回的是当前对象的时候 就不会调用intercept方法，相当于当前拦截器无效
     */
    Object plugin(Object target);

    /**
     * 用于在Mybatis配置文件中指定一些属性的，注册当前拦截器的时候可以设置一些属性
     */
    void setProperties(Properties properties);

}
~~~

#### 2.1.2 @Interceptor注解

Intercepts注解需要一个Signature(拦截点)参数数组。通过Signature来指定拦截哪个对象里面的哪个方法。@Intercepts/@signature注解定义如下:

~~~Java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Intercepts {
    /**
     * 定义拦截点
     * 只有符合拦截点的条件才会进入到拦截器
     */
    Signature[] value();
}
~~~

~~~Java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({})
public @interface Signature {
    /**
     * 定义拦截的类 Executor、ParameterHandler、StatementHandler、ResultSetHandler当中的一个
     */
    Class<?> type();

    /**
     * 在定义拦截类的基础之上，在定义拦截的方法
     */
    String method();

    /**
     * 在定义拦截方法的基础之上在定义拦截的方法对应的参数，
     * JAVA里面方法可能重载，不指定参数，不晓得是那个方法
     */
    Class<?>[] args();
}
~~~

示例:

~~~Java
@Intercepts({
        @Signature(
                type = Executor.class,// 拦截的类
                method = "query",// 方法
                args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class} // 参数
        ),
        @Signature(
                type = Executor.class,
                method = "update",
                args = {MappedStatement.class, Object.class}
        )
})
public class MybatisInterceptor implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {

        // TODO: 自定义拦截逻辑

    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this); // 返回代理类
    }

    @Override
    public void setProperties(Properties properties) {

    }
}

~~~

### 2.2 注册拦截器

注册拦截器就是去告诉mybatis去使用我们的拦截器,注册拦截器类非常简单,在@Configuration注解的类里面，@Bean我们自定义的拦截器类。比如我们需要注册自定义的MybatisInterceptor拦截器

~~~Java
/**
 * mybatis配置
 */
@Configuration
public class MybatisConfiguration {

    /**
     * 注册拦截器
     */
    @Bean
    public MybatisInterceptor mybatisInterceptor() {
        MybatisInterceptor interceptor = new MybatisInterceptor();
        Properties properties = new Properties();
        // 可以调用properties.setProperty方法来给拦截器设置一些自定义参数
        interceptor.setProperties(properties);
        return interceptor;
    }
    
    @Bean
    public PrepareInterceptor prepareInterceptor() {
        return  new PrepareInterceptor();
    }

}
~~~

通过注解注册也可以,方便快捷,在不需要传递自定义参数的时候可以使用

~~~Java
@Component
@Intercepts({
        @Signature(
                type = Executor.class,
                method = "query",
                args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}
        )
})
public class SelectPlugin implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        if (invocation.getTarget() instanceof Executor) {
            System.out.println("SelectPlugin");
        }
        return invocation.proceed();
    }

    @Override
    public Object plugin(Object target) {
        if (target instanceof Executor) {
            return Plugin.wrap(target, this);
        }
        return target;
    }

    @Override
    public void setProperties(Properties properties) {

    }
}
~~~



## 三.实例

### 3.1 拼装sql,用于数据权限控制

可以spring的aop搭配使用,登录时将关键权限数据储存至threadlocal或者权限验证的框架内,

### 3.2 日志打印

### 3.3 分页

### 3.4 分表

### 3.5 对查询结果某个字段加密

