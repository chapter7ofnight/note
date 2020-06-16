启动阶段

#### 一、获取 SqlSessionFactory(DefaultSqlSessionFactory) 的两种方式

##### 1）通过配置文件获取

​		读取配置文件到 Reader 或 InputStream 中，传递到 SqlSessionFactoryBuilder 对象里面，解析 XML 得到一个 Configuration 对象，最终通过这个 Configuration 对象构造出一个 SqlSessionFactory 的对象。

##### 2）通过 Java 代码获取

​		本质上和通过配置文件获取没有什么不同，只是把通过配置文件得到 Configuration 对象这一步移到代码里面，最终还是通过 Configuration 对象构造 SqlSessionFactory 对象。

##### 3）Mapper 文件和接口对应

​		（1）在 mappers 标签内配置 XML 文件的路径，然后通过 namespace 去指定对应的接口。

​		（2）配置接口的全类名，然后只会把全类名转换为路径去找 XML 文件。可以把 XML 放在 resources 下对应的目录内。也可以放在源码目录下，此时需要修改 maven 打包插件来 include 这些 XML。

​		不能同时使用以上两种方法定义同名的方法，因为 SQL 语句会被包装成一个 Statement 对象，从而放入一个 StrictMap 中，而这个 Map 的 key 就是方法的全名和简单名，也就是 XML 里面的 id。之后会把 接口 ---> MapperProxyFactory 的对应关系注册到 MapperRegistry。



运行阶段

#### 二、获取 SqlSession(DefaultSqlSession) 对象

​		这个对象不是线程安全的，所以每一次在不同的线程里执行 SQL 时，都需要重新从 SqlSessionFactory 中获取这个对象。

##### 1）获取到 Transaction 对象

​		通过 transactionManager 标签配置，默认为 ManagedTransaction。Transaction 是一个接口，所以允许用户根据需要自己进行**扩展**。

##### 2）获取 Executor 对象

​		这是实际执行 SQL 语句的对象，持有 Transaction 对象。这个接口有三个一般实现类：SimpleExecutor、ReuseExecutor 和 BatchExecutor（通过 defaultExecutorType 配置）。如果开启了二级缓存（通过 cacheEnabled 配置），还会使用 CachingExecutor 类来装饰，这是**装饰者模式**。（二级缓存需要在每个 XML 文件中配置 cache 节点，配合方法默认的 useCache 生效）

​		在 Executor 对象构造完成之后，会把通过 plugins 配置的插件（都实现了 Interceptor 接口，并用 @Intercepts 注解），依次应用于这个 Executor 上，这是**责任链模式**。注册的插件中指定的类如果有和通过 ExecutorType 选择的相同的类，则会为这个 Executor 类生成动态代理对象。在执行到通过 Intercepts 注解指定的方法后，会调用到 Interceptor.intercept 方法，这是需要自己实现的方法。这是**代理模式**。

##### 3）获取 SqlSession 对象

​		通过上面的构造获取到 DefaultSqlSession 对象。



#### 三、利用 SqlSession 执行 SQL

##### 1）获取 Mapper 对象

​		通过 SqlSession 从 MapperRegistry 中获取到 Mapper 对应的代理对象，当调用这个代理对象上的方法的时候，会在 MapperMethod 对象内根据不同的操作调到 SqlSession 上不同的数据库操作方法，并且做一些简单的返回值处理。这种不需要手动对参数进行包装，而由 ParamNameResolver 进行包装。

​		单个未注解的参数直接返回，一般用于传入一个对象。多个参数时，会按名字进行对应。

##### 2）直接调用方法

​		不获取 Mapper 对象，而是直接把要调用的方法名传给 SqlSession，从而调用相应的操作数据库的方法。需要自己把参数按规则包装。

##### 3.1）更新操作

​		这类操作包括 insert、delete 和 update，他们的特点是数据库都返回影响的行数。如果开启了二级缓存配置，会先确认是否需要刷新缓存。最终交给 BaseExecutor 去执行。BaseExecutor 定义抽象方法，而由子类去实现各个步骤，这是**模版方法模式**。

##### 3.2）查询操作

​		这类操作包括 select、selectList、selectMap、selectCursor，分别用来生成不同的返回值。前三种分别经过缓存判断后，交给 BaseExecutor 去执行。selectMap 还会自己转换返回类型为 Map。selectCursor 直接交给 BaseExecutor  执行。在 BaseExecutor 中实现了一级缓存。



#### 四、Executor 内部方法执行过程

##### 1）构建 StatementHandler 对象

​		又是**装饰者模式**，用 Routing**StatementHandler** 持有一个 PreparedStatementHandler（一般情况），再调用所有注册的插件。而在 debug 日志开启的情况下，Connection 和 Statement 都是使用**动态代理**生成的对象，在实际操作执行前，会打印日志。构建完成后会生成 Statement 对象（一般为 PrepareStatement）

##### 2）为 SQL 语句设置参数

​		Statement 对象构建完成后，会在准备阶段通过 **ParameterHandler** 把本次执行的操作的参数设置进去。

##### 3）通过 StatementHandler 对象执行操作

​		如果是 update 方法，直接执行操作，返回影响的行。同时需要额外处理生成主键的情况。

​		如果是 query 方法，执行操作之后，利用 **ResultSetHandler** 把原始返回转换成指定的类型。先通过默认构造方法去创建对象，或者只有一个构造方法，或者注解了 AutomapConstructor 的构造方法，或者其他和结果集可以匹配的构造方法。然后通过 resultType 的 get 方法去设置值，如果配置的是 resultMap，则会根据 resultMap 中的每个字段匹配值。