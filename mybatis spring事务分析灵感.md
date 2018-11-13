### spring mybatis sharding-jdbc事务分析



参考资料：

https://blog.csdn.net/chenwen_201116040110/article/details/46875135



起因：最近因为做分库分表用到了sharding-jdbc，在分库分表策略中将主表和子表的数据hash到了一个库上面，主要是为了利用本地事务来保证数据更改的事务特性，但是想到了spring的transaction标签在解析的时候sharding-jdbc内部还没有确定要连接到哪个datasource上，这样怎么能开启事务呢？于是就看了一下代码进行分析。下面主要是做一个总结：



一般常见的事务按我的理解分为三类：本地事务，弱事务，分布式事务。

1. 本地事务是指单库上面的事务，也就是用的最多的一种事务，有强一致性保证。
2. 弱事务，是指同时开启多个库的connection，做完逻辑后进行依次commit。这种可能出现多个connection中commit失败的场景，或者超时的场景等等，所以并不是很可靠，一致性并不是很强，并且有一直占用数据库资源的问题。但是一般场景下并没有问题，据了解有不少业务采用这种方式。
3. 分布式事务，是指应用层面上的一种事务，这种事务通过业务层面来保证一致性，并不会一直占用数据库层面的资源，commit和rollback都是在业务上进行解决。



这三种事务的话平时1和3用的比较多。比如前面提到的分库分表就是hash到同一个库保证本地事务。比如平时我们做强一致业务场景（比如支付等）的跨rpc调用也可能会用到分布式事务，毕竟有了网络的参与就带来更多的不确定性，所以需要利用分布式事务。sharding-jdbc目前看来是支持1和2的，据说3也会支持，所以我们今天主要是看看1和2是如何实现的。



介绍的大纲如下：

* 单独使用mybatis事务
* mybatis和spring结合使用，使用spring容器管理事务
* sharding-jdbc如何参与到spring和mybatis的事务中。


* ​

#### 单独使用mybatis事务

先看下使用的代码

```java
public static void main(String[] args) throws IOException {
		String resource = "mybatis-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);

        SqlSession sqlSession = sqlSessionFactory.openSession();
        try {
            TestMapper testMapper = sqlSession.getMapper(TestMapper.class);//1
            List<TestObject> dataList = testMapper.selectDataList();
 			System.out.println("ok");
        } finally {
            sqlSession.close();
        }
    }
```

上面是我们单独使用mybatis的时候用到的代码。可以看到，是通过sqlSession来操作的，关键操作是sqlSession.getMapper方法，通过这个方法创建出了一个TestMapper对象，









mybatis与spring结合的过程是通过创建mapper的代理对象，但是mapper的代理对象里面需要有sqlSession。所以spring创建了一个sqlSessionTemplate给所有的mapper代理对象。然后在mapper在调用自己的方法的时候就会通过invocationHandler（mapperproxy）采用反射的方式调用到对应的sqlSession的方法（比如selectOne，selectList）等等。

最终都会调用到sqlSessionTemplate这个类里面的方法，sqlSessionTemplate内部通过jdk的动态代理实现了一个sqlSessionInterceptor，这个方法就是最终所有逻辑的执行点！

```java
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      SqlSession sqlSession = getSqlSession(
          SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType,
          SqlSessionTemplate.this.exceptionTranslator);
      try {
        Object result = method.invoke(sqlSession, args);
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          // force commit even on non-dirty sessions because some databases require
          // a commit/rollback before calling close()
          sqlSession.commit(true);
        }
        return result;
      } catch (Throwable t) {
        Throwable unwrapped = unwrapThrowable(t);
        if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
          // release the connection to avoid a deadlock if the translator is no loaded. See issue #22
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
          sqlSession = null;
          Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
          if (translated != null) {
            unwrapped = translated;
          }
        }
        throw unwrapped;
      } finally {
        if (sqlSession != null) {
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
      }
    }
```

可以看到，实际上每次执行的时候都会去getSqlSession，在getSqlSession里面我们具体看：

```java
  public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {

    notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);
    notNull(executorType, NO_EXECUTOR_TYPE_SPECIFIED);

    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);

    SqlSession session = sessionHolder(executorType, holder);
    if (session != null) {
      return session;
    }

    if (LOGGER.isDebugEnabled()) {
      LOGGER.debug("Creating a new SqlSession");
    }

    session = sessionFactory.openSession(executorType);

    registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);

    return session;
  }
```

先通过TransactionSynchronizationManager来看一下当前线程是否有sqlSession这个对象，如果有的话就直接返回，否则创建一个新的sqlSession，并且将它绑定到当前线程，这样再执行就可以获取相同的sqlSession。



这里实际上是通过threadlocal的方式来管理sqlSession，这样保证了sqlSession的线程安全，同时能够保证不会每次执行都new一个sqlSession对象。



sqlSession在被创建的时候需要传入executor，executor在创建的时候需要创建transaction，根据代码分析如下

```java
    if (this.transactionFactory == null) {
      this.transactionFactory = new SpringManagedTransactionFactory();
    }

    configuration.setEnvironment(new Environment(this.environment, this.transactionFactory, this.dataSource));

```

这里使用的springManagedTransaction。



mybatis的executor在具体执行某个方法的时候会调用transaction来获取连接。而在spring中，这个transaction就是springManagedTransaction。他的openConnection方法如下：

```java
  private void openConnection() throws SQLException {
    this.connection = DataSourceUtils.getConnection(this.dataSource);
    this.autoCommit = this.connection.getAutoCommit();
    this.isConnectionTransactional = DataSourceUtils.isConnectionTransactional(this.connection, this.dataSource);

    if (LOGGER.isDebugEnabled()) {
      LOGGER.debug(
          "JDBC Connection ["
              + this.connection
              + "] will"
              + (this.isConnectionTransactional ? " " : " not ")
              + "be managed by Spring");
    }
  }
```

可以看到获取连接不是像我们平时单独使用mybatis的jdbcTransaction的时候一样从datasource直接获取一个链接，而是调用了DataSourceUtils的getConnection方法。如下

```java
	public static Connection doGetConnection(DataSource dataSource) throws SQLException {
		Assert.notNull(dataSource, "No DataSource specified");

		ConnectionHolder conHolder = (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);
		if (conHolder != null && (conHolder.hasConnection() || conHolder.isSynchronizedWithTransaction())) {
			conHolder.requested();
			if (!conHolder.hasConnection()) {
				logger.debug("Fetching resumed JDBC Connection from DataSource");
				conHolder.setConnection(dataSource.getConnection());
			}
			return conHolder.getConnection();
		}
		// Else we either got no holder or an empty thread-bound holder here.

		logger.debug("Fetching JDBC Connection from DataSource");
		Connection con = dataSource.getConnection();

		if (TransactionSynchronizationManager.isSynchronizationActive()) {
			logger.debug("Registering transaction synchronization for JDBC Connection");
			// Use same Connection for further JDBC actions within the transaction.
			// Thread-bound object will get removed by synchronization at transaction completion.
			ConnectionHolder holderToUse = conHolder;
			if (holderToUse == null) {
				holderToUse = new ConnectionHolder(con);
			}
			else {
				holderToUse.setConnection(con);
			}
			holderToUse.requested();
			TransactionSynchronizationManager.registerSynchronization(
					new ConnectionSynchronization(holderToUse, dataSource));
			holderToUse.setSynchronizedWithTransaction(true);
			if (holderToUse != conHolder) {
				TransactionSynchronizationManager.bindResource(dataSource, holderToUse);
			}
		}

		return con;
	}
```

发现调用了TransactionSynchronizationManager的方法，如果继续跟进去就会发现是采用threadlocal的方式保存了当前线程的使用连接，所以springManagedTransaction的策略是如果当前线程已经有了连接就不进行新建连接了，否则从dataSource里面获取connection，并且看需要是否保存到当前线程的threadlocal里面。



上面的分析说明了spring与mybatis的对接过程中有两个非常重要的工作，第一就是通过mapper接口来创建代理对象，并且注册到spring容器中。另外就是mybatis完全让spring进行事务管理，spring通过继承了platformTransactionManager的类dataSourceTransactionManager在mybatis外部管理事务。具体由于我们一般都采用的transactional标签，所以在transactionInterceptor类里面会做具体的事务处理，处理的逻辑如下：

```java
protected Object invokeWithinTransaction(Method method, Class<?> targetClass, final InvocationCallback invocation)
			throws Throwable {

		// If the transaction attribute is null, the method is non-transactional.
		final TransactionAttribute txAttr = getTransactionAttributeSource().getTransactionAttribute(method, targetClass);
		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
		final String joinpointIdentification = methodIdentification(method, targetClass);

		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
			// Standard transaction demarcation with getTransaction and commit/rollback calls.
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
			Object retVal = null;
			try {
				// This is an around advice: Invoke the next interceptor in the chain.
				// This will normally result in a target object being invoked.
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// target invocation exception
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
				cleanupTransactionInfo(txInfo);
			}
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}
	}
```

简化后的代码如上，可以看到首先调用了createTransactionIfNecessary方法。这个方法通过名字看就是判断是否需要创建transaction的，因为spring的事务有几种传播级别。

1. Never：如果当前有事务的话就抛出异常

   Mandatory：如果当前有事务就加入事务，如果没有的话抛出异常

2. Required：如果当前有事务的话就加入事务，如果没有的话就新建事务

   Required_New：如果当前有事务就创建一个新的事务并且挂起当前事务，如果没有的话就新建事务。实际上是创建一个完全新的事务，和老的事务没有关系。

3. Not_supported：如果当前有事务的话就挂起当前事务，然后执行。

   Supports：如果当前有事务的话就加入事务，如果没有的话就忽略

4. Nested：实际上是创建一个savepoint，如果nested的事务失败了就回滚到上一个savepoint，它只是父亲事务的一部分。

一共有七种事务的传播级别，但是我们可以分成四类来记忆，其中比较难以区分的就是RequiredNew和Nest的区别。其实nested还是在当前事务中执行，只不过创建了savepoint，但是RequiredNew就是新建一个事务了。

由上面分析可知，在解析transaction标签的时候，spring的事务管理器就获取了一个connection了，所以我们要尽量避免长事务以及将rpc调用放在事务里面执行。































