---
title: Spring事务管理
date: 2016-10-25 20:40:09
tags: Spring
---
# Spring事务管理

Spring框架为事务管理提供了一致的抽象，可以带来以下好处：

- Java Transaction API（JTA），JDBC，Hibernate，Java Persistence API（JPA）和Java数据对象（JDO）等不同的事务API之间的一致的编程模型
- 支持声明事务管理
- 编程式事务管理的API使用简单
- 与Spring的数据访问抽象集成

## Spring一致的编程模型

传统上，Java EE开发人员有两种事务管理选择：全局事务或本地事务，两者都有明显的局限性。

- 全局事务
 JTA、EJB
> 优点：可以多资源使用;
>
> 缺点：JTA API笨重、通过JNDI获取资源。

- 本地事务

本地事务是资源专用，比如：JDBC连接。
> 优点：简单易用;
>
>缺点：不能多资源使用。

### 理解Spring事务抽象

Spring事务管理SPI(Service Provider Interface)的抽象层主要包括3个接口，分别是PlatformTransactionManager、TransactionDefinition和TransactionStatus，它们位于org.springframework.transaction包中。

![Spring事务管理SPI抽象](http://i4.piimg.com/598959/c19eb0b638b35414.png)

PlatformTransactionManager根据TransactionDefinition提供的事务配置信息创建事务，并用TransactionStatus描述这个激活事务的状态。

#### TransactionDefinition

TransactionDefinition用于描述事务的隔离级别、超时时间、是否只读事务和事务传播规则等控制事务具体行为的事务属性。

```java
public interface TransactionDefinition {

    // 事务传播
    int getPropagationBehavior();

    // 事务隔离级别
    int getIsolationLevel();

    // 事务超时事务
    int getTimeout();

    // 是否只读
    boolean isReadOnly();

    // 事务名称
    String getName();

}
```

##### 事务隔离级别

当前事务和其他事务的隔离程度。在TransactionDefinition接口中，定义了和java.sql.Connection接口中同名的4个隔离级别：ISOLATION_READ_UNCOMMITTED、ISOLATION_READ_COMMITTED、ISOLATION_REPEATABLE_READ和ISOLATION_SERIALIZABLE。此外，TransactionDefinition还定义了一个默认的隔离级别——ISOLATION_DEFAULT,它表示使用底层数据库的默认隔离级别。

##### 事务传播

Spring通过事务传播行为控制当前的事务如何传播到被嵌套调用的目标服务接口方法中。

Spring在TransactionDefinition接口中规定了7种类型的事务传播行为，它们规定了事务方法和事务方法发生嵌套调用时事务如何进行传播，如下表：

| 事务传播行为类型                  | 说明                                                        |
|---------------------------|-----------------------------------------------------------|
| PROPAGATION_REQUIRED      | 如果当前没有事务，则新建一个事务；如果已经存在一个事务，则加入到这个事务中。这是最常见的选择。           |
| PROPAGATION_SUPPORTS      | 支持当前事务。如果当前没有事务，则以非事务方式执行                                 |
| PROPAGATION_MANDATORY     | 使用当前事务。如果当前没有事务，则抛出异常                                     |
| PROPAGATION_REQUIRES_NEW  | 新建事务。如果当前存在事务，则把当前事务挂起                                    |
| PROPAGATION_NOT_SUPPORTED | 以非事务方式执行。如果当前存在事务，则抛出异常                                   |
| PROPAGATION_NEVER         | 以非事务方式执行。如果当前存在事务，则抛出异常                                   |
| PROPAGATION_NESTED        | 如果当前存在事务，则在嵌套事务内执行；如果当前没有事务，则执行与PROPAGATION_REQUIRED类似的操作 |

#### TransactionStatus

TransactionStatus代表一个事务的具体运行状态。事务管理器可以通过该接口获取事务运行期的状态信息，也可以通过该接口间接地回滚事务，它相比于抛出异常时回滚事务的方式更具可控性。该接口继承于SavepointManager接口，SavepointManager接口基于JDBC3.0保存点的分段事务控制能力提供嵌套事务的机制。

```java
public interface TransactionStatus extends SavepointManager, Flushable {

    // 返回当前事务状态是否是新事务
    boolean isNewTransaction();

    // 返回当前事务是否有保存点；
    boolean hasSavepoint();

    // 设置当前事务应该回滚；
    void setRollbackOnly();

    // 返回当前事务是否应该回滚；
    boolean isRollbackOnly();

    // 用于刷新底层会话中的修改到数据库，一般用于刷新如Hibernate/JPA的会话，可能对如JDBC类型的事务无任何影响；
    void flush();

    // 当前事务否已经完成。
    boolean isCompleted();

}
```

#### PlatformTransactionManager

Spring框架支持事务管理的核心是事务管理器抽象，对于不同的数据访问框架（如Hibernate）通过实现策略接口PlatformTransactionManager，从而能支持各种数据访问框架的事务管理。

##### PlatformTransactionManager接口定义

```java
public interface PlatformTransactionManager {

    // **根据指定传播行为，获取当前存在的事务或者创建一个新的事务。**
    TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;

    // **提交事务**
    void commit(TransactionStatus status) throws TransactionException;

    // **事务回滚**
    void rollback(TransactionStatus status) throws TransactionException;
}
```

##### Spring的事务管理器实现类

Spring将事务管理委托给底层具体的持久化实现框架来完成。因此，Spring为不同的持久化框架提供了PlatformTransactionManager接口的实现类。如下表所示：

事务                                                             | 说明
-----------------------------------------------------------------|-----------------------------------------------------------------------------------
org.springframework.orm.jpa.JpaTransactionManager                | 使用JPA进行持久化，使用该事务管理器
org.springframework.orm.hibernateX.HibernateTransactionManager   | 使用Hibernate X.0(X可为3,4,5)版本进行持久化时，使用该事务管理器
org.springframework.jdbc.datasource.DataSourceTransactionManager | 使用Spring JDBC或MyBatis等基于DataSource数据源技术的持久化技术时，使用该事务管理器
org.springframework.orm.jdo.JdoTransactionManager                | 使用JDO进行持久化时，使用该事务管理器
org.springframework.orm.jpa.JpaTransactionManager                | 具有多个数据源的全局事务使用该事务管理器（不管采用何种持久化技术）

## 基本用法

三种方式：编程、XML配置和注解。第一方式对应用代码侵入性较大，现已较少使用。后面两种则都属于声明式事务管理的方式，两者的共同点是都提供事务管理信息的元数据，只不过方式不同。前者对代码的侵入性最小，也最为常用，后者则属于较为折衷的方案，有一点侵入性，但相对也较少了配置，各有优劣，依场景需求而定。声明式事务管理是Spring的一大亮点，利用AOP技术将事务管理作为切面动态织入到目标业务方法中，让事务管理简单易行。

  而不管是使用哪种方式，数据源、事务管理器都是必须的，一般通过XML的Bean配置：

```xml
<bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource">
    <property name="driverClass" value="com.mysql.jdbc.Driver"/>
    <property name="jdbcUrl" value="jdbc:mysql://localhost:3306/test"/>
    <property name="user" value="root"/>
    <property name="password" value="root"/>
</bean>

<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
    <property name="dataSource" ref="dataSource"/>
</bean>

<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
```

### 编程式事务管理

采用与DAO模板类一样的开闭思想，Spring提供了线程安全的TransactionTemplate模板类来处理不变的事务管理逻辑，将变化的部分抽象为回调接口TransactionCallback供用户自定义数据访问逻辑。使用示例：

```java
public class UserServiceImpl implements UserService {

    @Resource
    private JdbcTemplate jdbcTemplate;

    @Resource
    private TransactionTemplate transactionTemplate;

    @Override
    public void save(User user) {
        transactionTemplate.execute(new TransactionCallbackWithoutResult() {
            @Override
            protected void doInTransactionWithoutResult(TransactionStatus status) {
                jdbcTemplate.update("INSERT INTO user(name,age,sex) VALUES (?,?,?)",
                        new Object[]{user.getName(), user.getAge(), user.getSex()},
                        new int[]{Types.VARCHAR, Types.INTEGER, Types.VARCHAR});

                throw new RuntimeException();
            }
        });
    }
}
```

TransactionTemplate的配置信息

```xml
<bean id="transactionTemplate" class="org.springframework.transaction.support.TransactionTemplate">
    <property name="transactionManager" ref="txManager"/>
</bean>
```

### 基于XML Schema配置的事务管理

```xml
<tx:advice id="txAdvice" transaction-manager="txManager">
    <tx:attributes>
        <tx:method name="get*" read-only="true"/>
        <tx:method name="*"/>
    </tx:attributes>
</tx:advice>


<aop:config>
    <aop:pointcut id="serviceOperation"
            expression="execution(* com.codersm.study.spring.tx.service.UserService.*(..))"/>
    <aop:advisor advice-ref="txAdvice" pointcut-ref="serviceOperation"/>
</aop:config>
```

### 基于注解的事务管理

通过@Transactional对需要事务增强的Bean接口、实现类或方法进行标注，在容器中配置<tx:annotation-driven>以启用基于注解的声明式事务。

```java
@Transactional
public class UserServiceImpl implements UserService {

    @Resource
    private JdbcTemplate jdbcTemplate;

    @Override
    public void save(User user) {
        jdbcTemplate.update("INSERT INTO user(name,age,sex) VALUES (?,?,?)",
                new Object[]{user.getName(), user.getAge(), user.getSex()},
                new int[]{Types.VARCHAR, Types.INTEGER, Types.VARCHAR});

    }
}
```

## Spring事务管理源码分析

> 源码分析一定要有目的性，至少有一条清晰的主线，比如要搞清楚框架的某一个功能点背后的代码组织，前因后果，而不是一头扎进源码里，无的放矢。本文就从Spring事务管理的三种使用方式入手，逐个分析Spring在背后都为我们做了些什么。

### 编程式

#### TransactionTemplate

TransactionTemplate是编程式事务管理的入口，源码如下：

```java
public <T> T execute(TransactionCallback<T> action) throws TransactionException {
    if (this.transactionManager instanceof CallbackPreferringPlatformTransactionManager) {
        return ((CallbackPreferringPlatformTransactionManager) this.transactionManager)
                            .execute(this, action);
    }
    else {
        TransactionStatus status = this.transactionManager.getTransaction(this);【1】
        T result;
        try {
            result = action.doInTransaction(status);
        }
        catch (RuntimeException ex) {
            // Transactional code threw application exception -> rollback
            rollbackOnException(status, ex);【2】
            throw ex;
        }
        catch (Error err) {
            // Transactional code threw error -> rollback
            rollbackOnException(status, err);
            throw err;
        }
        catch (Throwable ex) {
            // Transactional code threw unexpected exception -> rollback
            rollbackOnException(status, ex);
            throw new UndeclaredThrowableException(ex, 
                "TransactionCallback threw undeclared checked exception");
        }
        this.transactionManager.commit(status);【3】
        return result;
    }
}
```

TransactionTemplate提供了唯一的编程入口execute，它接受用于封装业务逻辑的TransactionCallback接口的实例，返回用户自定义的事务操作结果T。具体逻辑：先是判断transactionManager是否是接口CallbackPreferringPlatformTransactionManager的实例，若是则直接委托给该接口的execute方法进行事务管理；否则交给它的核心成员PlatformTransactionManager进行事务的创建、提交或回滚操作。

> 可以看出基于TransactionTemplate的事务管理，在发生RuntimeException、Error或Exception时都会回滚，正常时才提交事务。

CallbackPreferringPlatformTransactionManger接口扩展自PlatformTransactionManger，根据以下的官方源码注释可知，该接口相当于是把事务的创建、提交和回滚操作都封装起来了，用户只需要传入TransactionCallback接口实例即可，而不是像使用PlatformTransactionManger接口那样，还需要用户自己显示调用getTransaction、rollback或commit进行事务管理。

#### 具体剖析

【1】获取事务

```java
public final TransactionStatus getTransaction(TransactionDefinition definition)
         throws TransactionException {

    Object transaction = doGetTransaction();【*】

    // Cache debug flag to avoid repeated checks.
    boolean debugEnabled = logger.isDebugEnabled();

    if (definition == null) {
        // Use defaults if no transaction definition given.
        definition = new DefaultTransactionDefinition();
    }

    if (isExistingTransaction(transaction)) { 【*】
        // Existing transaction found -> check propagation behavior to find out how to behave.
        return handleExistingTransaction(definition, transaction, debugEnabled);【*】
    }

    // Check definition settings for new transaction.
    if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
        throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
    }

    // No existing transaction found -> check propagation behavior to find out how to proceed.
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
        throw new IllegalTransactionStateException(
                "No existing transaction found for transaction marked with propagation 'mandatory'");
    }
    else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
            definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
            definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        SuspendedResourcesHolder suspendedResources = suspend(null);【*】
        if (debugEnabled) {
            logger.debug("Creating new transaction with name"
                    +"[" + definition.getName() + "]: " + definition);
        }
        try {
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            DefaultTransactionStatus status = newTransactionStatus(
                    definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
            doBegin(transaction, definition);【*】
            prepareSynchronization(status, definition);【*】
            return status;
        }
        catch (RuntimeException ex) {
            resume(null, suspendedResources);【*】
            throw ex;
        }
        catch (Error err) {
            resume(null, suspendedResources);
            throw err;
        }
    }
    else {
        // Create "empty" transaction: no actual transaction, but potentially synchronization.
        if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
            logger.warn("Custom isolation level specified but no actual transaction initiated; " +
                    "isolation level will effectively be ignored: " + definition);
        }
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
    }
}
```

判断事务对象是否存在，如果存在，则根据定义的事务传播属性进行相应的处理返回TransactionStatus对象；不存的且需要事务，则创建一个TransactionStatus对象，获取数据源Connection，将Connection设置为不自动提交事务，存储在ThreadLocal中,同步线程相关联的资源。

【2】事务回滚

```java
private void rollbackOnException(TransactionStatus status, Throwable ex) throws TransactionException {
    this.transactionManager.rollback(status);
}

public final void rollback(TransactionStatus status) throws TransactionException {
    DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
    processRollback(defStatus);
}

private void processRollback(DefaultTransactionStatus status) {
    try {
        try {
            triggerBeforeCompletion(status);
            if (status.hasSavepoint()) {
                status.rollbackToHeldSavepoint();
            }
            else if (status.isNewTransaction()) {
                doRollback(status);
            }
            else if (status.hasTransaction()) {
                if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
                    doSetRollbackOnly(status);
                }
                else {
                    if (status.isDebug()) {
                        logger.debug("Participating transaction failed "
                                +"- letting transaction originator decide on rollback");
                    }
                }
            }
            else {
                logger.debug("Should roll back transaction but cannot - no transaction available");
            }
        }
        triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
    }
    finally {
        cleanupAfterCompletion(status);
    }
}


protected void doRollback(DefaultTransactionStatus status) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
    Connection con = txObject.getConnectionHolder().getConnection();
    if (status.isDebug()) {
        logger.debug("Rolling back JDBC transaction on Connection [" + con + "]");
    }
    try {
        con.rollback();
    }
    catch (SQLException ex) {
        throw new TransactionSystemException("Could not roll back JDBC transaction", ex);
    }
}
```

【3】事务提交

```java
public final void commit(TransactionStatus status) throws TransactionException {
    if (status.isCompleted()) {
        throw new IllegalTransactionStateException(
                "Transaction is already completed - do not call commit or rollback more than once per transaction");
    }

    DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
    if (defStatus.isLocalRollbackOnly()) {
        if (defStatus.isDebug()) {
            logger.debug("Transactional code has requested rollback");
        }
        processRollback(defStatus);
        return;
    }
    if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
        if (defStatus.isDebug()) {
            logger.debug("Global transaction is marked as rollback-only but transactional code requested commit");
        }
        processRollback(defStatus);
        // Throw UnexpectedRollbackException only at outermost transaction boundary
        // or if explicitly asked to.
        if (status.isNewTransaction() || isFailEarlyOnGlobalRollbackOnly()) {
            throw new UnexpectedRollbackException(
                    "Transaction rolled back because it has been marked as rollback-only");
        }
        return;
    }

    processCommit(defStatus);
}

private void processCommit(DefaultTransactionStatus status) throws TransactionException {
    try {
        boolean beforeCompletionInvoked = false;
        try {
            prepareForCommit(status);
            triggerBeforeCommit(status);
            triggerBeforeCompletion(status);
            beforeCompletionInvoked = true;
            boolean globalRollbackOnly = false;
            if (status.isNewTransaction() || isFailEarlyOnGlobalRollbackOnly()) {
                globalRollbackOnly = status.isGlobalRollbackOnly();
            }
            if (status.hasSavepoint()) {
                if (status.isDebug()) {
                    logger.debug("Releasing transaction savepoint");
                }
                status.releaseHeldSavepoint();
            }
            else if (status.isNewTransaction()) {
                if (status.isDebug()) {
                    logger.debug("Initiating transaction commit");
                }
                doCommit(status);
            }
            // Throw UnexpectedRollbackException if we have a global rollback-only
            // marker but still didn't get a corresponding exception from commit.
            if (globalRollbackOnly) {
                throw new UnexpectedRollbackException(
                        "Transaction silently rolled back because it has been marked as rollback-only");
            }
        }
        catch (UnexpectedRollbackException ex) {
            // can only be caused by doCommit
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
            throw ex;
        }
        catch (TransactionException ex) {
            // can only be caused by doCommit
            if (isRollbackOnCommitFailure()) {
                doRollbackOnCommitException(status, ex);
            }
            else {
                triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
            }
            throw ex;
        }
        catch (RuntimeException ex) {
            if (!beforeCompletionInvoked) {
                triggerBeforeCompletion(status);
            }
            doRollbackOnCommitException(status, ex);
            throw ex;
        }
        catch (Error err) {
            if (!beforeCompletionInvoked) {
                triggerBeforeCompletion(status);
            }
            doRollbackOnCommitException(status, err);
            throw err;
        }

        // Trigger afterCommit callbacks, with an exception thrown there
        // propagated to callers but the transaction still considered as committed.
        try {
            triggerAfterCommit(status);
        }
        finally {
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
        }

    }
    finally {
        cleanupAfterCompletion(status);
    }
}

protected void doCommit(DefaultTransactionStatus status) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
    Connection con = txObject.getConnectionHolder().getConnection();
    if (status.isDebug()) {
        logger.debug("Committing JDBC transaction on Connection [" + con + "]");
    }
    try {
        con.commit();
    }
    catch (SQLException ex) {
        throw new TransactionSystemException("Could not commit JDBC transaction", ex);
    }
}
```

### 声明式

基于XML和注解的方式都是属于声明式事务管理，只是提供元数据的形式不用。声明式事务的核心实现就是利用AOP技术，将事务逻辑作为环绕增强MethodInterceptor动态织入目标业务方法中。其中的核心类为TransactionInterceptor。

#### TransactionInterceptor

```java
/**
* AOP Alliance MethodInterceptor for declarative transaction management using the common Spring transaction  infrastructure (PlatformTransactionManager).
*/
public class TransactionInterceptor extends TransactionAspectSupport
     implements MethodInterceptor, Serializable
```

它的核心方法invoke如下：

```java
public Object invoke(final MethodInvocation invocation) throws Throwable {

    Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

    return invokeWithinTransaction(invocation.getMethod(), targetClass, new InvocationCallback() {
        @Override
        public Object proceedWithInvocation() throws Throwable {
            return invocation.proceed();
        }
    });
}

protected Object invokeWithinTransaction(Method method, Class<?> targetClass,
     final InvocationCallback invocation) throws Throwable {

    // If the transaction attribute is null, the method is non-transactional.
    final TransactionAttribute txAttr = getTransactionAttributeSource().getTransactionAttribute(method, targetClass);
    final PlatformTransactionManager tm = determineTransactionManager(txAttr);
    final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

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

    else {
        // It's a CallbackPreferringPlatformTransactionManager: pass a TransactionCallback in.
        try {
            Object result = ((CallbackPreferringPlatformTransactionManager) tm).execute(txAttr,
                    new TransactionCallback<Object>() {
                        @Override
                        public Object doInTransaction(TransactionStatus status) {
                            TransactionInfo txInfo = prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
                            try {
                                return invocation.proceedWithInvocation();
                            }
                            catch (Throwable ex) {
                                if (txAttr.rollbackOn(ex)) {
                                    // A RuntimeException: will lead to a rollback.
                                    if (ex instanceof RuntimeException) {
                                        throw (RuntimeException) ex;
                                    }
                                    else {
                                        throw new ThrowableHolderException(ex);
                                    }
                                }
                                else {
                                    // A normal return value: will lead to a commit.
                                    return new ThrowableHolder(ex);
                                }
                            }
                            finally {
                                cleanupTransactionInfo(txInfo);
                            }
                        }
                    });

            // Check result: It might indicate a Throwable to rethrow.
            if (result instanceof ThrowableHolder) {
                throw ((ThrowableHolder) result).getThrowable();
            }
            else {
                return result;
            }
        }
        catch (ThrowableHolderException ex) {
            throw ex.getCause();
        }
    }
}
```

TransactionInterceptor实现了MethodInterceptor接口，将事务管理的逻辑封装在环绕增强的实现中，而目标业务代码则抽象为MethodInvocation（该接口扩展自Joinpoint，故实际是AOP中的连接点），使得事务管理代码与业务逻辑代码完全分离，可以对任意目标类进行无侵入性的事务织入。具体逻辑：先根据MethodInvocation获取事务属性TransactionAttribute，根据TransactionAttribute得到对应的PlatformTransactionManager，再根据其是否是CallbackPreferringPlatformTransactionManager的实例分别做不同的处理，整体上跟TransactionTemplate中大相径庭，后面主要是介绍几点不同的地方。

#### 具体剖析

##### MethodInterceptor

MethodInterceptor是AOP中的环绕增强接口，同一个连接点可以有多个增强，而TransactionInterceptor扩展自该接口，说明事务管理只是众多横切逻辑中的一种。

##### TransactionAttribute和TransactionAttributeSource

```java
public interface TransactionAttribute extends TransactionDefinition {

    /**
    * Return a qualifier value associated with this transaction attribute.
    * <p>This may be used for choosing a corresponding transaction manager
    * to process this specific transaction.
    */
    String getQualifier();

    /**
    * Should we roll back on the given exception?
    * @param ex the exception to evaluate
    * @return whether to perform a rollback or not
    */
    boolean rollbackOn(Throwable ex);

}
```

就是在TransactionDefinition的基础上增加了两个可定制属性。

```java
/**
 * Strategy interface used by {@link TransactionInterceptor} for metadata retrieval.
 *
 * <p>Implementations know how to source transaction attributes, whether from configuration,
 * metadata attributes at source level (such as Java 5 annotations), or anywhere else.
 */
public interface TransactionAttributeSource {

    TransactionAttribute getTransactionAttribute(Method method, Class<?> targetClass);
}
```

TransactionAttributeSource获取事务元数据的抽象接口，不同的来源对应不同实现类。

![TransactionAttributeSource具体实现类](http://i2.kiimg.com/598959/cfcae098bf3b6abd.png)

##### TransactionAspectSupport

事务切面基本抽象实现类，其提供了如下方法：
![TransactionAspectSupport提供的方法](http://i1.buimg.com/598959/3e4ded89a2ab3a18.png)

## 其他

### 获取数据连接资源

当脱离模板类，直接操作底层持久技术的原生API时，就需要通过Spring提供的资源工具类获取线程绑定的资源，而不应该直接从DataSource或SessionFactory中获取，否则容易造成数据连接泄露的问题。Spring为不同的持久化技术提供了一套从TransactionSynchronizationManager中获取对应线程绑定资源的工具类：DataSourceUtils（Spring JDBC或iBatis）、SessionFactoryUtils（Hibernate 3.0）等。

### 如何标注@Transactional注解

虽然@Transactional注解可被应用于接口、接口方法、类及类的public方法，但建议在具体实现类上使用@Transactional注解，因为接口上的注解不能被继承，这样会有隐患（关于注解的继承，可参考这里）。当事务配置按如下方式，使用的是子类代理（CGLib）而非接口代理（JDK）时，对应目标类不会添加事务增强！

### Using @Transactional with AspectJ

It is also possible to use the Spring Framework’s @Transactional support outside of a Spring container by means of an AspectJ aspect.

### 事务绑定事件

As of Spring 4.2, the listener of an event can be bound to a phase of the transaction.The typical example is to handle the event when the transaction has completed successfully: this allows events to be used with more flexibility when the outcome of the current transaction actually matters to the listener.

## 参考资料

1. Spring官方文档
2. 精通Spring4.x企业应用开发实战
3. [Spring事务管理](http://www.jianshu.com/p/80778d9d11f3)
