## 1.jdbc方式查询数据库存在的问题：

> -     数据库连接频繁创建、释放，浪费资源，影响性能
> -     sql语句、参数设置、结果解析硬编码不易维护

## 2.自定义框架设计思路

> | 问题               | 思路                                       |
> | ------------------ | ------------------------------------------ |
> | 频繁创建、释放连接 | 使用连接池                                 |
> | sql硬编码          | 使用xml配置文件                            |
> | 参数设置硬编码     | 使用占位符动态替换参数                     |
> | 解析返回硬编码     | xml维护字段与对象字段映射关系,使用反射内省 |


## 3.使用动态代理为mapper接口生成代理对象
> sqlSession.getMapper(XXX.class);
 ```java
 @Override
 public <T> T getMappper(Class<?> mapperClass) {
 T o = (T) Proxy.newProxyInstance(mapperClass.getClassLoader(), new Class[]
 {mapperClass}, new InvocationHandler() {
 @Override
 public Object invoke(Object proxy, Method method, Object[] args)
 throws Throwable {
             String methodName = method.getName();
             String className = method.getDeclaringClass().getName();
             //使用className+methodName作为statementId，
             //要求xml中namespace和id值与之对应
             String key = className+"."+methodName;
             MappedStatement mappedStatement =
             configuration.getMappedStatementMap().get(key);
             //获取statement后根据标签类型调用select 或者 update
             //如果是select的话需要判断方法返回值类型是否存在泛型。如果是则为selectList，否为selectOne；
             Type genericReturnType = method.getGenericReturnType();
             if(genericReturnType instanceof ParameterizedType){
                 return selectList();
             }else{
                 return selectOne
             }
        }
     }
 }
 ```

## 4.mybatis 使用配置
```xml
<environments default='dev'>
    <environment id="dev">
        <transactionManager type="JDBC"/>
        <dataSource type="POOLED">
            <property name="driver" value="${jdbc.drive}"
            ……
        </dataSource>
    </environment>
</environments>
```
### transactionManager配置方式有两种
- JDBC：直接使用jdbc的提交和回滚设置，依赖于数据源
- MANAGED：几乎没有做什么。从来不提交或回滚一个连接。


### 数据源类型dataSource有三种类型

- UNPOLLED：不使用连接池，每次都创建释放连接
- POOLED：使用连接池
- JNDI：为了能再入EJB活应用服务器这类容器中使用，可以集中或在外部配置数据源，然后防止一个JNDI上下文引用


### 5.mybatis 注解开发

| 注解     | 作用                               |
| -------- | ---------------------------------- |
| @Insert  | 插入                               |
| @Select  | 查询                               |
| @Delete  | 删除                               |
| @Update  | 更新                               |
| @Result  | 结果集封装                         |
| @Results | 可与@Result结合使用 封装多个结果集 |
| @One     | 一对一结果封装                     |
| @Many    | 一对多结果封装                     |

实例：
```java
public interface UserMapper {
@Select("select * from user")
@Results({
@Result(id = true,property = "id",column = "id"),
@Result(property = "username",column = "username"),
@Result(property = "password",column = "password"),
@Result(property = "birthday",column = "birthday"),
@Result(property = "orderList",column = "id",
javaType = List.class,
many = @Many(select =
"com.lagou.mapper.OrderMapper.findByUid"))
 })
List<User> findAllUserAndOrder();
}
```

## 6.一二级缓存

######  a).存储结构:
> 一级缓存：使用PerpetualCache对象中的hashMap存储，在执行sql前根据查询信息生成CacheKey对象，从缓中查询是否已经有之前保存的查询结果，如果有则直接返回，如果没有则到数据中查询相关数据并put到cache中。默认开启

> 二级缓存：默认也使用PerpetualCache对象中的hashMap存储，可由用户指定使用其他方式存储，如redis，需要实现mybatis的Cache接口，并且需要在使用时制定使用的缓存类。多个SqlSession可以共用二级缓存，二级缓存是跨SqlSession。默认不开启

######  b).范围:

> 一级缓存：作用域是SqlSession，

> 二级缓存：mapper（namespace）级别的缓存

######  c).失效场景:
> 一级缓存：当查询到的数据，进行增删改的操作的时候，缓存将会失效

> 二级缓存：如果在相同的namespace下的mapper映射文件中增删改，并且提交了事务，就会失效。

二级缓存开启方式：

需要现在sqlMapConfig.xml配置中开启
```xml
<settings> 
    <setting name="cacheEnabled" value="true"/>
</settings>
```
其次需要在mapper文件中开启<cache></cache>

默认使用的是 PerpetualCache，也可以使用其他实现了cache接口的类，如使用redis作为mybatis的二级缓存，则在配置时将需要使用的缓存类设置进去
```xml
<cache type="org.mybatis.caches.redis.RedisCache" />
```

## 7.mybatis 插件
### mybatis支持用插件对四大核心对象进行拦截，从而实现功能的增强。
四大核心对象为

| 对象             | 说明          |
| ---------------- | ------------- |
| Executor         | 执行器        |
| StatementHandler | sql语法构建器 |
| ParameterHandler | 参数处理器    |
| ResultSetHandler | 结果集处理器  |

### 插件的原理
四大对象都不是直接返回的，而是使用代理的方式。

### 插件使用举例
1.需要实现Interceptor接口
2.为类添加需要拦截的方法签名
3.讲插件类配置到configration中

```java
@Intercepts({
        @Signature(type = Executor.class, method = "query", args = {
                MappedStatement.class, Object.class, RowBounds.class,
                ResultHandler.class}),
        @Signature(type = Executor.class, method = "update", args = {
                MappedStatement.class, Object.class})
})
public DomeInteceptor implements Interceptor{
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
       //实现intercept方法从而达到功能增强
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {

    }
}
```

config文件中增加
```xml
<plugins> 
    <plugin interceptor="DomeInteceptor"></plugin>
</plugins>
```

优秀的mybatis插件
> - 1.pageHelper 对返回接口进行分页封装
> - 2.通用mapper 对于单表查询不需要编写增删改查sql，由插件提供相应实现

### 8.mybatis 工作流程
- 加载配置文件并初始化
- 接收调用请求，实现参数的处理。（sql处理，参数设置）
- 获取数据库连接执行查询，释放连接
- 根据配置解析返回结果集