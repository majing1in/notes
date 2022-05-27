> **官方文档地址**
>
> **https://mybatis.org/mybatis-3/zh/index.html**

# 一、Mybatis源码解析

## 1、Mybatis加载流程

---

1. 在 new SqlSessionFactoryBuilder().build(inputStream) 的 build() 方法中创建 XML 配置生成器
2. 在 XMLConfigBuilder 构造函数中会对 Configuration 进行初始化，并初始化 XPathParser 解析器用于解析配置文件中的标签
3. 最终在 parseConfiguration 方法中解析 mybatis-config.xml 配置文件

```java
// 读取Mybatis配置文件
InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
// 构建SqlSessionFactory实例
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(inputStream);
```

---

1. parseConfiguration 方法中的配置信息基本都会放入到 Configuration 类中
2. 其中包含重要的数据库连接配置，之后会通过这些配置创建 Statement 操作数据库
3. 当解析完 mappers 标签后，Configuration 会管理各个 mapper.xml 

```java
private void parseConfiguration(XNode root) {
    try {
        propertiesElement(root.evalNode("properties"));
        Properties settings = settingsAsProperties(root.evalNode("settings"));
        loadCustomVfs(settings);
        loadCustomLogImpl(settings);
        // 配置驼峰转换
        typeAliasesElement(root.evalNode("typeAliases"));
        // 装载插件
        pluginElement(root.evalNode("plugins"));
        // 对象工厂
        objectFactoryElement(root.evalNode("objectFactory"));
        objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
        reflectorFactoryElement(root.evalNode("reflectorFactory"));
        // 解析 setting 标签
        settingsElement(settings);
        // 数据库环境配置
        environmentsElement(root.evalNode("environments"));
        databaseIdProviderElement(root.evalNode("databaseIdProvider"));
        typeHandlerElement(root.evalNode("typeHandlers"));
        // 解析 mappers 标签中配置的各个 xml
        mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
        throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
}
```

---

重点方法 mapperElement(root.evalNode("mappers")) ，这段代码逻辑简单，它会解析 mappers 中所有的 mapper 标签，之后对每一个配置的 mapper 进行循环解析，其中 "resource"、"url" 直接导入的 xml 文件，class 导入的是 mapper 关联的接口

**[映射器（mappers）三种资源定位方式](#1、映射器（mappers）三种资源定位方式)**

```java
private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
        for (XNode child : parent.getChildren()) {
            if ("package".equals(child.getName())) {
                String mapperPackage = child.getStringAttribute("name");
                configuration.addMappers(mapperPackage);
            } else {
                String resource = child.getStringAttribute("resource");
                String url = child.getStringAttribute("url");
                String mapperClass = child.getStringAttribute("class");
                if (resource != null && url == null && mapperClass == null) {
                    ErrorContext.instance().resource(resource);
                    InputStream inputStream = Resources.getResourceAsStream(resource);
                    XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
                    mapperParser.parse();
                } else if (resource == null && url != null && mapperClass == null) {
                    ErrorContext.instance().resource(url);
                    InputStream inputStream = Resources.getUrlAsStream(url);
                    XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
                    mapperParser.parse();
                } else if (resource == null && url == null && mapperClass != null) {
                    Class<?> mapperInterface = Resources.classForName(mapperClass);
                    configuration.addMapper(mapperInterface);
                } else {
                    throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
                }
            }
        }
    }
}
```

---

Mybatis加载流程核心方法

1. 判断当前的 mapper.xml是否加载，如果未加载则进行加载
2. 解析当前的 xml 文件并加入到 Configuration 中防止重复加载
3. 绑定接口与 xml 之间的关系（反射实现）
4. Mybatis 在处理 xml 配置文件时，是按照从上到下顺序进行解析的，但是在解析一个节点的时候，该节点可能引用后面定义的还未解析的节点，遇到这种情况时，Mybatis 会抛出对应的 IncompleteExementException异常。根据抛出异常的节点不同，Mybatis 会创建不同的解析对象，并添加到 Configuration 对象中暂存，便于后续再次处理，parsePendingResultMaps、parsePendingCacheRefs、parsePendingStatements 方法便是用于处理这种情况。

```java
public void parse() {
    if (!configuration.isResourceLoaded(resource)) {
        // 解析 mapper.xml 中各个标签
        configurationElement(parser.evalNode("/mapper"));
        configuration.addLoadedResource(resource);
        // 绑定接口与 mapper.xml 的关系
        bindMapperForNamespace();
    }
    parsePendingResultMaps();
    parsePendingCacheRefs();
    parsePendingStatements();
}
```

**解析 mapper.xml 中的标签**

1. 先获取到 namespace 放入到全局配置中
2. 解析缓存标签、结果映射标签、select|insert|update|delete标签

```java
private void configurationElement(XNode context) {
    try {
        // 获取到 namespace，之后通过全限定名发生获取对应的 Class 对象
        String namespace = context.getStringAttribute("namespace");
        if (namespace == null || namespace.isEmpty()) {
            throw new BuilderException("Mapper's namespace cannot be empty");
        }
        builderAssistant.setCurrentNamespace(namespace);
        // 二级缓存标签
        cacheRefElement(context.evalNode("cache-ref"));
        cacheElement(context.evalNode("cache"));
        // 已过时的方式
        parameterMapElement(context.evalNodes("/mapper/parameterMap"));
        // 解析结果集映射
        resultMapElements(context.evalNodes("/mapper/resultMap"));
        sqlElement(context.evalNodes("/mapper/sql"));
        /**
        	1.构建每一个真正执行的 SQL 标签语句
        	2.方法会构建一个 MappedStatement 对象，保存了这条SQL的所有信息（SQL source、Cache等）
        	3.最终这个 MappedStatement 对象会被保存到 Configuration 的 Map 中，其中 key 是 				  namespace + id，value 是当前的 statement 对象
        **/
        buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
        throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
    }
}
```

**绑定接口与 Mapper 之间的关系**

1. 通过 namespace 权限定名获取到对应的 Mapper 接口
2. 将获取的 Class 对象放入到 MapperRegistry 对象中
3. MapperRegistry 对象维护一个 Map 对象，其中 key 为当前的Class对象，value 为 MapperProxyFactory 封装的自定义代理工厂对象

```java
private void bindMapperForNamespace() {
    String namespace = builderAssistant.getCurrentNamespace();
    if (namespace != null) {
        Class<?> boundType = null;
        try {
            boundType = Resources.classForName(namespace);
        } catch (ClassNotFoundException e) {
            // ignore, bound type is not required
        }
        if (boundType != null && !configuration.hasMapper(boundType)) {
            // Spring may not know the real resource name so we set a flag
            // to prevent loading again this resource from the mapper interface
            // look at MapperAnnotationBuilder#loadXmlResource
            configuration.addLoadedResource("namespace:" + namespace);
            configuration.addMapper(boundType);
        }
    }
}
```

## 2、Mybatis查询过程

**创建一个查询的 SQL Session 对象**

1. 根据 Configuration 全局配置类对象获取到数据库连接信息 Environment，最终获取到 Transaction 对象(包装数据库连接，处理连接生命周期，包括：其创建、准备、提交回滚和关闭)
2. 通过传入的 Transaction 与 执行器类型生成不同的执行器，最终构建 SqlSession 并返回

```java

private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
        final Environment environment = configuration.getEnvironment();
        final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
        tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
        final Executor executor = configuration.newExecutor(tx, execType);
        return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
        closeTransaction(tx); // may have fetched a connection so lets call close()
        throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```

**通过 SqlSession 获取 Mapper 对象**

1. 在 MapperRegistry 中保存了所有的 Mapper 对象，利用传入的 Class 直接获取到 MapperProxyFactory 对象
2. MapperProxyFactory 在构建的时候已经保存了接口信息( JDK 动态代理基于接口代理)
3. 利用 JDK 动态代理生成代理对象

```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
        throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
        return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
        throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
}

public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
}

protected T newInstance(MapperProxy<T> mapperProxy) {
    // mapperProxy 实际生成的代理对象
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
}
```

**调用代理方法**

当调用代理对象中的方法时，会进入到具体的 InvocationHandler 实现类中，执行它的 invoke 方法(基本的动态代理)

```java
// 执行 MmapperProxy 方法中的 invoke 方法
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
        if (Object.class.equals(method.getDeclaringClass())) {
            return method.invoke(this, args);
        } else {
            return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
        }
    } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
    }
}
// 最终调用的方法
private static class PlainMethodInvoker implements MapperProxy.MapperMethodInvoker {
    private final MapperMethod mapperMethod;
    public PlainMethodInvoker(MapperMethod mapperMethod) {
        super();
        this.mapperMethod = mapperMethod;
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args, SqlSession sqlSession) throws Throwable {
        return mapperMethod.execute(sqlSession, args);
    }
}
```

**执行具体的方法**

1. 根据 SQL 类型执行不同的逻辑(select|insert|update|delete)
2. 以查询为基础最终都会执行到  selectList  中
3. 通过 statement = namespace + id 方式获取到 MapperStatement 对象

```java
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
        // 增加
        case INSERT: {
			...
            break;
        }
		// 修改
        case UPDATE: {
			...
            break;
        }
		// 删除
        case DELETE: {
            ...
            break;
        }
		// 查询
        case SELECT:
            ...
            break;
        case FLUSH:
            result = sqlSession.flushStatements();
            break;
        default:
            ...
    }
    return result;
}

private <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler) {
    try {
        // 通过加载时保存的 Mapper.xml 中具体的查询方法
        MappedStatement ms = configuration.getMappedStatement(statement);
        return executor.query(ms, wrapCollection(parameter), rowBounds, handler);
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```

**查询逻辑**

1. 生成 CacheKey 对象，由 namespace、id、SQL语句等组成，保证唯一性(一二级缓存)
2. 根据 CacheKey 从一级缓存中获取缓存值，否则就查数据库
3. 根据 Configuration 对象构建 StatementHandler 对象操作数据库
4. 查询数据库并将结果放入到以及缓存中

```java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
        clearLocalCache();
    }
    List<E> list;
    try {
        queryStack++;
        // 先从一级缓存中获取值
        list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
        // 不为空则直接获取否则查询数据库
        if (list != null) {
            handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
        } else {
            list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
        }
    } finally {
        queryStack--;
    }
    if (queryStack == 0) {
        for (DeferredLoad deferredLoad : deferredLoads) {
            deferredLoad.load();
        }
        // issue #601
        deferredLoads.clear();
        if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
            // issue #482
            clearLocalCache();
        }
    }
    return list;
}

// 查询数据库
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
        list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
        localCache.removeObject(key);
    }
    // 将查询的结果放入到一级缓存中
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
        localOutputParameterCache.putObject(key, parameter);
    }
    return list;
}
// 查询数据库方法
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
        Configuration configuration = ms.getConfiguration();
        StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
        stmt = prepareStatement(handler, ms.getStatementLog());
        // 获取到 Statement 对象后操作数据库查询结果
        return handler.query(stmt, resultHandler);
    } finally {
        closeStatement(stmt);
    }
}
// 构建操操作数据库的 Statement 对象
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    Connection connection = getConnection(statementLog);
    stmt = handler.prepare(connection, transaction.getTimeout());
    handler.parameterize(stmt);
    return stmt;
}
```

## 3、一级缓存

## 4、二级缓存

## 5、插件原理

## 6、结果集映射原理

## 7、动态SQL原理

# 二、Mybatis使用示例

## 1、映射器（mappers）三种资源定位方式

```xml
<!-- 使用相对于类路径的资源引用 -->
<mappers>
  <mapper resource="org/mybatis/builder/AuthorMapper.xml"/>
  <mapper resource="org/mybatis/builder/BlogMapper.xml"/>
  <mapper resource="org/mybatis/builder/PostMapper.xml"/>
</mappers>
<!-- 使用完全限定资源定位符（URL） -->
<mappers>
  <mapper url="file:///var/mappers/AuthorMapper.xml"/>
  <mapper url="file:///var/mappers/BlogMapper.xml"/>
  <mapper url="file:///var/mappers/PostMapper.xml"/>
</mappers>
<!-- 使用映射器接口实现类的完全限定类名 -->
<mappers>
  <mapper class="org.mybatis.builder.AuthorMapper"/>
  <mapper class="org.mybatis.builder.BlogMapper"/>
  <mapper class="org.mybatis.builder.PostMapper"/>
</mappers>
<!-- 将包内的映射器接口实现全部注册为映射器 -->
<mappers>
  <package name="org.mybatis.builder"/>
</mappers>
```

