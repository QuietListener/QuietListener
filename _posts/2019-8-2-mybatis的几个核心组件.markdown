---
layout: post
title:  mybatis的几个核心组件
date:   2019-8-2 14:32:00
categories: mybatis orm mysql
---

需要知道的几个组件
1. Transaction(JdbcTransaction和ManagedTransaction)
2. TransactionFactory(JdbcTransactionFactory和ManagedTransactionFactory)
3. SqlSession
4. SqlSessionFactory
5. Executor 执行sql

## 1.Transaction
###1. mybatis使用Transaction来管理事务
```
public interface Transaction { 
 Connection getConnection() throws SQLException;
 void commit() throws SQLException;
 void rollback() throws SQLException;
 void close() throws SQLException;
 Integer getTimeout() throws SQLException;
}
```

###2. 两种Transaction实现:
JdbcTransaction和ManagedTransaction
JdbcTransaction 自己管理commit和rollback
ManagedTransaction的commit和rollback是空的由容器自己来管理(spring)，如下所示
```
@Override
public void commit() throws SQLException {
 // Does nothing
}

@Override
public void rollback() throws SQLException {
 // Does nothing
}
```
3.transaction commit调用其实是
transaction.commit->connection.commit




## 2.TransactionFactory
Transaction是由TransactionFactory来实例化的

### 1.JdbcTransactionFactory和ManagedTransactionFactory
TransactionFactory有两个实现：
JdbcTransactionFactory，生成JdbcTransaction
ManagedTransactionFactory，生成ManagedTransaction)

### 2. 关系
TransactionFactory生成Transaction，Transaction管理 connection的commit和rollback

### 3. 配置TransactionFactory

使用 \<transactionManager type="JDBC"/>标签来指定 用jdbc还是managed。

```
  <environment id="db1">
       <transactionManager type="JDBC"/>
       <dataSource type="POOLED">
      …..
</environment>
```



## 3. SqlSession
SqlSession是程序员编程接触最多的东西。
```public static  User query(int id)
{
   //从SqlSessionFactory获取一个SqlSession
   SqlSession session = sqlSessionFactory.openSession();
   try{
       //下面是具体业务
       UserMapper mapper = session.getMapper(UserMapper.class);
       User user = mapper.selectUser(id);=
       user.setName(" abc " + new Date());
       mapper.insert(user);

       //提交
       session.commit();
       return user;
   }
   catch(Exception e)
   {
       e.printStackTrace();
       //回滚
       session.rollback();
   }
   finally {
       //关闭
       session.close();
   }
```
### 1.SqlSessionFactory生成SqlSession
**DefaultSqlSessionFactory是SqlSessionFactory的子类，用来生成SqlSession**
```
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
 Transaction tx = null;
 try {
   final Environment environment = configuration.getEnvironment();

//生成TransactionFactory
final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);

 //TransactionFactory生成Transaction
 tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);

 //以Transaction为参数生成Executor
 final Executor executor = configuration.newExecutor(tx, execType);

 // 以/executor为参数生成DefaultSqlSession
 return new DefaultSqlSession(configuration, executor, autoCommit);

 } catch (Exception e) {
   closeTransaction(tx); // may have fetched a connection so lets call close()
   throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
 } finally {
   ErrorContext.instance().reset();
 }
}

```
####过程如下
1. 获取对应的TransactionFactory，并且生成transaction
2. 以transaction为参数生成**Executor**，
```
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
 executorType = executorType == null ? defaultExecutorType : executorType;
 executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
 Executor executor;
 if (ExecutorType.BATCH == executorType) {
   executor = new BatchExecutor(this, transaction);
 } else if (ExecutorType.REUSE == executorType) {
   executor = new ReuseExecutor(this, transaction);
 } else {
   executor = new SimpleExecutor(this, transaction);
 }
 if (cacheEnabled) {
   executor = new CachingExecutor(executor);
 }
 executor = (Executor) interceptorChain.pluginAll(executor);
 return executor;
}
```
3. 以executor为参数生成 sqlSession
return new DefaultSqlSession(configuration, executor, autoCommit);

**所以sqlSession调用commit其实是调用下面过程**
SqlSession.commit->Executor.commit->transaction.commit->connection.commit



## 4.SqlsessionFactory

### 1.builid SqlsessionFactory
需要配置文件和需要对应一个environment的名字比如”db1“
```
public SqlSessionFactory build(InputStream inputStream, String environment) {
 return build(inputStream, environment, null);
}
```

### 2.SqlSessionFactory的构建
需要指定TransactionFactory(**\<transactionManager type="JDBC"/>**)和一个Datasource  

可以在配置文件的environment中指定了 TransactionFactory和DataSource
```
<environments default="development">
   <environment id="db1">
       <transactionManager type="JDBC"/>
       <dataSource type="POOLED">
           <property name="driver" value="com.mysql.jdbc.Driver"/>
           <property name="url"
                     value="jdbc:mysql://localhost:3306/mybatis-test"/>
           <property name="username" value="root"/>
           <property name="password" value=""/>
       </dataSource>
   </environment>

   <environment id="db2">
       <transactionManager type="JDBC"/>
       <dataSource type="POOLED">
           <property name="driver" value="com.mysql.jdbc.Driver"/>
           <property name="url"
                     value="jdbc:mysql://localhost:3306/mybatis-test-1"/>
           <property name="username" value="root"/>
           <property name="password" value=""/>
       </dataSource>
   </environment>
</environments
```

