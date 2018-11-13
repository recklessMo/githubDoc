### mybatis核心要点分析



创建sqlSessionFactory

创建sqlSession

通过sqlSession执行sql-》实际上内部都是通过executor来执行具体的逻辑

executor里面有transaction对象



```java
mybatis的核心api是通过sqlSession来进行操作的，在创建sqlsession的时候会传入一个executor，sqlsession在执行sql的过程中通过调用executor来执行逻辑。executor在执行sql的的时候需要获取连接，获取连接是通过transaction来获取的，所以executor在创建的时候会传入transaction对象，transaction里面保留了connection或者datasource对象，因为在获取连接的时候需要设置autoCommit或者transactionIsolationLevel等属性。之后executor用获取的connection来创建Statement，创建完statement之后就进入了正常的jdbc执行流程了。
所以核心逻辑就是sqlSession的执行！
```







mybatis平时用的很多，但是对原理性的东西却缺乏了解。最近在做分库分表的过程中遇到了事务问题，正好借这个机会了解了一下mybatis的原理。

核心关键点：

1. 初始化
2. mapper接口动态代理
3. ​





mybatis使用的流程是：

```java
public static void main(String[] args) throws IOException {
		String resource = "mybatis.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);

        SqlSession sqlSession = sqlSessionFactory.openSession();
        try {
            ProductMapper productMapper = sqlSession.getMapper(ProductMapper.class);//1
            List<Product> productList = productMapper.selectProductList();
            for (Product product : productList) {
                System.out.printf(product.toString());
            }
        } finally {
            sqlSession.close();
        }
    }
```

核心入口是sqlSession，sqlSessionFactory来创建sqlSession。



1. 比较magic的地方就是1这个地方，这里sqlSession获取了一个mapper，这里很明显是通过jdk的动态代理构建了一个实例对象，因为ProductMapper本身是一个接口。所以这里就是核心地方。



```java
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);//1
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      return mapperProxyFactory.newInstance(sqlSession);//2
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```



1. mybatis启动的时候会先扫描所有的mapper，然后放入knownMappers这个对象之中。
2. 通过mapperProxyFactory来构建出一个productMapper来。



```java
  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }

  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);//1
  }

```

1. 可以看到，是通过jdk的动态代理的方式来创建了一个代理，代理对应的invocationHandler就是MapperProxy这个类。所以实际上调用productMapper的时候是调用了MapperProxy里面的方法。



```java
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {//1
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);//2
    return mapperMethod.execute(sqlSession, args);//3
  }

```

1. 首先排除掉object里面的方法，别把那些基础方法也代理了
2. 从缓存里面拿对应的MapperMethod对象，如果对象没有的话就新建
3. 通过MapperMethod来执行对应的productMapper里面的方法。



```java
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
      case INSERT: {
    	Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));//1
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));//1
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));//1
        break;
      }
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
          if (method.returnsOptional() &&
              (result == null || !method.getReturnType().equals(result.getClass()))) {
            result = Optional.ofNullable(result);
          }
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName() 
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
```

1. 可以看到这里调用了sqlSession的基础方法，所以采用动态代理这种方式，绕了一大圈，实际上就回到了最初的利用sqlSession的地方。



所以从头开始：

现在mybatis的执行就变成了：通过xml里面配置的sql语句加上参数来执行。如下：

```java
  @Override
  public int update(String statement, Object parameter) {
    try {
      dirty = true;
      MappedStatement ms = configuration.getMappedStatement(statement);//1
      return executor.update(ms, wrapCollection(parameter));//2
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error updating database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

1. 通过sql语句的id去配置里面取对应的MappedStatement
2. 通过executor来执行对应的MappedStatement



我们以最简单的SimpleExecutor来举例

```java
  @Override
  public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);//1
      stmt = prepareStatement(handler, ms.getStatementLog());//2
      return handler.update(stmt);
    } finally {
      closeStatement(stmt);
    }
  }
```

1. 通过StatementHandler来执行对应的statement
2. 需要创建对应的statement。分为三种statement





















