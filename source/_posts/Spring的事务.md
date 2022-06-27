---
title: Spring 的事务
categories:  
	- [JAVA后端开发 , Spring]
tags: 
	- Spring
	- 学习笔记
date: 2018-12-25 00:00:00
---

# Spring 的事务

## Spring支持的事务策略
Java EE应用的传统事务有两种策略：全局事务和局部事务。全局事务由应用服务器管理，需要底层服务器的JTA(全称Java Transaction API ，即Java事务API)。局部事务和底层所采用的持久化技术相关，当采用JDBC持久化技术时，需要使用Connection对象来操作事务；而采用Hibernate持久化技术时，需要使用Session对象来操作事务。

1. 术语
> - 全局事务：Global Transactions
> - 局部事务：Local Transactions
> - JTA：Java Transaction API(Java事务接口)
> - CMT：Container Managed Transaction(容器管理事务)
> - 声明式事务管理：Declarative Transaction Management
> - 编程式事务管理：Programmatic Transaction Management
> - JNDI：Java Naming and Directory Interface(Java命名和目录接口)


2. 全局事务
    全局事务由应用服务器通过JTA进行管理。以前，使用全局事务比较流行的方法是采用EJB CMT，CMT是声明式事务管理的一种形式（区别于编程式事务管理）。尽管使用EJB本身就需要使用JNDI ，EJB CMT不需要事务相关的JNDI lookups。EJB CMT不需要编写大量的Java代码来控制事务。使用CMT最大的不足之处是它被束缚在JTA和应用服务器上。CMT只有当在EJB中实现业务逻辑时才可以发挥作用，或者至少处于事务性EJB Façade中。由于EJB存在诸多的不足并且目前存在其他可选的声明式事务管理的解决方案，所以不太建议使用EJB方式。

    1. 全局事务的优点：
        - 全局事务支持多个事务性资源间的相互工作（如：关系型数据库和消息队列）。
    2. 全局事务的缺点：
        1. 采用全局事务需要使用JTA，而JTA是一个笨重的API。
        2. 通常情况下，JTA UserTransaction需要从JNDI获取。这意味着，如果我们使用JTA，就需要同时使用JTA和JNDI。
        3. 通常JTA只能在应用服务器环境下使用，因此使用JTA会限制代码的复用性。
3. 局部事务
    局部事务是资源特有的（resource-specific），最常见的例子是与JDBC连接相关联的事务。使用局部事务，应用服务器不参与事务管理并且不能保证访问多个资源的正确性。值得注意的是：大多数应用程序使用单个的事务性资源。   
    1. 局部事务的优点：
        - 易用
    2. 局部事务的缺点：
        1. 局部事务不支持多个事务性资源间的相互工作，比如：使用JDBC连接来管理事务的代码不能在全局JTA事务上运行。
        2. 局部事务趋向于编程模型，编程模型具有侵入性。

Spring可以使应用程序开发者在任何环境下使用统一的编程模型。应用程序开发者只需要编写一次代码，就可以使用不同环境下的不同事务管理策略。Spring既支持声明式事务管理，也支持编程式事务管理。

我们推荐使用声明式事务管理。采用编程式事务管理，开发者需要使用Spring提供的事务抽象（它可以在任何基础的事务架构上运行）。采用声明式事务管理，开发者仅需要编写少量甚至不需要编写与事务管理相关的代码，因此它不依赖于Spring的事务API。

Spring事务策略是通过PlatformTransactionManage接口体现的，该接口是Spring事务策略的核心，该接口定义如下：
``` java
public interface PlatformTransactionManage
{
    // 平台无关的获得事务的方法
    TransactionStatus getTransaction(TransactionDefinition definition) 
    throws TransactionException;
    //平台无关的事务提交方法
    void commit(TransactionStatus status) throws TransactionException;
    //平台无关的事务回滚方法
    void rollback(TransactionStatus status) throws TransactionException;
}
```

PlatformTransactionManage是一个与任何事务策略分离的接口，随着低层不同事务策略的切换，应用必须采用不同的实现类。

PlatformTransactionManage接口有许多不同的实现类，应用程序面向与平台无关的接口编程，当底层采用不同的持久层技术时，系统只需使用不同的PlatformTransactionManage实现类即可————而这种切换通常由Spring容器负责管理，应用程序既无须与具体的事务API耦合，也无须与特定实现类耦合，从而将应用和持久化技术、事务API彻底分离处理。

Spring的事务机制是一种典型的策略模式，PlatformTransactionManage代表事务管理接口，但它并不知道底层到底如何管理事务，它只要求事务管理需要提供三个事务方法，但具体如何实现则交给其实现类来完成————不同实现则代表不同的事务管理策略。

即使使用容器管理的JTA，代码也依然无须指定JNDI查找，无须与特定的JTA资源耦合在一起，通过配置文件，JTA资源传给PlatformTransactionManage的实现类。因此，程序的代码可在JTA事务管理和非JTA事务管理之间轻松切换。

-------

在PlatformTransactionManage接口内，包含一个`getTransaction(TransactionDefinition definition)`方法，该方法根据TransactionDefinition参数返回一个TransactionStatus对象。TransactionStatus对象表示一个事务，TransactionStatus被关联在当前执行的线程上。

`getTransaction(TransactionDefinition definition)`返回的TransactionStatus对象，可能是一个新的事务，也可能是一个已经存在的事务对象。如果当前执行的线程已经处于事务管理下，则返回当前线程的事务对象；否则，系统将新建一个事务对象后返回。

TransactionDefinition接口定义了事务规则，该接口必须制定如下几个属性值：
> + 事务隔离：当前事务和其他事务的隔离程度
>   1. TransactionDefinition.ISOLATION_DEFAULT
>       这是默认值，表示使用底层数据库的默认隔离级别。对大部分数据库而言，通常这值就是TransactionDefinition.ISOLATION_READ_COMMITTED
>   2. TransactionDefinition.ISOLATION_READ_UNCOMMITTED
>       该隔离级别表示一个事务可以读取另一个事务修改但还没有提交的数据。该级别不能防止脏读，不可重复读和幻读，因此很少使用该隔离级别。比如PostgreSQL实际上并没有此级别。
>   3. TransactionDefinition.ISOLATION_READ_COMMITTED
>       该隔离级别表示一个事务只能读取另一个事务已经提交的数据。该级别可以防止脏读，这也是大多数情况下的推荐值。
>   4. TransactionDefinition.ISOLATION_REPEATABLE_READ
>       该隔离别表示一个事务在整个过程中可以多次重复执行某个查询，并且每次返回的记录都相同。该级别可以防止脏读和不可重复读。
>   5. TransactionDefinition.ISOLATION_SERIALIZABLE
>       所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，该级别可以防止脏读、不可重复读以及幻读。但是这将严重影响程序的性能。通常情况下也不会用到该级别。
>   
>   > MYSQL: 默认为REPEATABLE_READ级别
>   > SQLSERVER: 默认为READ_COMMITTED
>   > + 脏读 ：一个事务读取到另一事务未提交的更新数据
>   > + 不可重复读 ：在同一事务中, 多次读取同一数据返回的结果有所不同, 换句话说, 后续读取可以读到另一事务已提交的更新数据. 相反, "可重复读"在同一事务中多次读取数据时, 能够保证所读数据一样, 也就是后续读取不能读到另一事务已提交的更新数据
>   > + 幻读 ：一个事务读到另一个事务已提交的insert数据。

> + 事务传播：通常，在事务中执行的代码都会在当前事务中运行
>   1. TransactionDefinition.PROPAGATION_REQUIRED
>   如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。这是默认值。
>   2. TransactionDefinition.PROPAGATION_REQUIRES_NEW
>   创建一个新的事务，如果当前存在事务，则把当前事务挂起
>   3. TransactionDefinition.PROPAGATION_SUPPORTS
>   如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
>   4. TransactionDefinition.PROPAGATION_NOT_SUPPORTED
>   以非事务方式运行，如果当前存在事务，则把当前事务挂起。
>   5. TransactionDefinition.PROPAGATION_NEVER
>   以非事务方式运行，如果当前存在事务，则抛出异常。
>   6. TransactionDefinition.PROPAGATION_MANDATORY
>   如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。
>   7. TransactionDefinition.PROPAGATION_MANDATORY
>   如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。
> 
> + 事务超时：指一个事务所允许执行的最长时间，如果超过该时间限制但事务还没有完成，则自动回滚事务。在 TransactionDefinition 中以 int 的值来表示超时时间，其单位是秒。默认设置为底层事务系统的超时值，如果底层数据库事务系统没有设置超时值，那么就是none，没有超时限制。
> 
> + 事务只读属性：只读事务用于客户代码只读但不修改数据的情形，只读事务用于特定情景下的优化，比如使用Hibernate的时候。默认为读写事务

TransactionStatus代表事务本省，它提供了简单的控制事务执行和查询事务状态的方法，这些方法在所有事务API中都是相同的。TransactionStatus接口的定义如下：
``` java
public interface TransactionStatus
{
    // 判断事务是否为新建的事务
    boolean isNewTransaction();
    // 设置事务回滚
    void setRollbackOnly();
    // 查询事务是否已有回滚标志
    boolean isRollbackOnly();
}
```

## 使用XML Schema配置事务策略
Spring提供了tx:命名空间来配置事务管理，tx:命名空间下提供了`<tx:advice.../>`元素来配置事务增强处理，一旦使用该元素配置了事务管理处理，就可以直接使用`<aop:advisor.../>`元素启用自动代理了。

配置`<tx:advice.../>`时除了需要transaction-manager属性指定事务管理器之外，还需要配置一个`<attributes.../>`子元素，该子元素里又可包含多个`<method.../>`子元素。

`<method.../>`子元素可以指定如下属性：
> name：必选属性，与该事务语义关联的方法名。该属性支持通配符
> propagation：指定事务传播行为
> isolation：指定事务隔离级别
> timeout：指定事务超时时间
> read-only：指定事务是否只读
> rollback-for：指定触发时间回滚的异常类，该属性可指定多个异常类，多个异常类用逗号隔开
> no-rollback-for：指定不触发时间回滚的一场了，该属性可指定多个异常类，多个异常类用逗号隔开

`<aop:advisor>`作用：将Advice和切入点绑定在一起，保证Advice所包含的增强处理将在对应的切入点被织入。当采用`<aop:advisor>`元素将Advice和切入点绑定时，实际上是由Spring提供的Bean后处理器完成的。Spring提供了BeanNameAutoProxyCreator、DefaultAdvisorAutoProxyCreator两个Bean后处理器。

实例代码：
NewsDaoImpl组件包含一个insert()方法，该方法同时插入两条记录，但插入的第二条记录将会违反唯一约束，从而应发异常
``` java 
public class NewsDaoImpl implements NewsDao
{
	private DataSource ds;
	public void setDs(DataSource ds)
	{
		this.ds = ds;
	}
	public void insert(String title, String content)
	{
		JdbcTemplate jt = new JdbcTemplate(ds);
		jt.update("insert into news_inf"
			+ " values(null , ? , ?)"
			, title , content);
		// 两次插入的数据违反唯一键约束
		jt.update("insert into news_inf"
			+ " values(null , ? , ?)"
			, title , content);
		// 如果没有事务控制，则第一条记录可以被插入
		// 如果增加事务控制，将发现第一条记录也插不进去。
	}
}
```
配置文件：
``` xml
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns="http://www.springframework.org/schema/beans"
	xmlns:p="http://www.springframework.org/schema/p"
	xmlns:aop="http://www.springframework.org/schema/aop"
	xmlns:tx="http://www.springframework.org/schema/tx"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans.xsd
	http://www.springframework.org/schema/aop
	http://www.springframework.org/schema/aop/spring-aop.xsd
	http://www.springframework.org/schema/tx
	http://www.springframework.org/schema/tx/spring-tx.xsd">
	<!-- 定义数据源Bean，使用C3P0数据源实现，并注入数据源的必要信息 -->
	<bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource"
		destroy-method="close"
		p:driverClass="com.mysql.jdbc.Driver"
		p:jdbcUrl="jdbc:mysql://localhost/spring?useSSL=true"
		p:user="root"
		p:password="32147"
		p:maxPoolSize="40"
		p:minPoolSize="2"
		p:initialPoolSize="2"
		p:maxIdleTime="30"/>
	<!-- 配置JDBC数据源的局部事务管理器，使用DataSourceTransactionManager 类 -->
	<!-- 该类实现PlatformTransactionManager接口，是针对采用数据源连接的特定实现-->
	<!-- 配置DataSourceTransactionManager时需要依注入DataSource的引用 -->
	<bean id="transactionManager" 
		class="org.springframework.jdbc.datasource.DataSourceTransactionManager"
		p:dataSource-ref="dataSource"/>
	<!-- 配置一个业务逻辑Bean -->
	<bean id="newsDao" class="org.crazyit.app.dao.impl.NewsDaoImpl"
		p:ds-ref="dataSource"/>
	<!-- 配置事务增强处理Bean,指定事务管理器 -->
	<tx:advice id="txAdvice" 
		transaction-manager="transactionManager">
		<!-- 用于配置详细的事务语义 -->
		<tx:attributes>
			<!-- 所有以'get'开头的方法是read-only的 -->
			<tx:method name="get*" read-only="true" timeout="8"/>
			<!-- 其他方法使用默认的事务设置，指定超时时长为5秒 -->
			<tx:method name="*" isolation="DEFAULT"
				propagation="REQUIRED" timeout="5"/>
		</tx:attributes>
	</tx:advice>
	<!-- AOP配置的元素 -->
	<aop:config>
		<!-- 配置一个切入点，匹配org.crazyit.app.dao.impl包下
			所有以Impl结尾的类里、所有方法的执行 -->
		<aop:pointcut id="myPointcut"
			expression="execution(* org.crazyit.app.dao.impl.*Impl.*(..))"/>
		<!-- 指定在myPointcut切入点应用txAdvice事务增强处理 -->
		<aop:advisor advice-ref="txAdvice" 
			pointcut-ref="myPointcut"/>
	</aop:config>
</beans>
```
运行上面的程序，将出现一个异常，而且insert()方法所执行的两条SQL语句全部回滚

## 使用@Transactional
@Transactional可用于修饰Spring Bean类，也可用于修饰Bean类中的某个方法。如果使用@Transactional修饰Bean类，则表明这些食物设置对整个Bean起作用；如果使用@Transactional修饰Bean类的某个方法，则表明这些事务设置只对该方法有效。

使用@Transactional可指定如下属性：
> - isolation：用于指定事务的隔离级别。默认为底层事务的隔离级别
> - noRollbackFor：指定遇到特定异常时强制不回滚事务
> - noRollbackForClassName：指定遇到特定的多个异常时强制不回滚事务。该属性值可以指定多个异常名。
> - propagation：指定事务传播行为
> - readonly：指定事务是否只读
> - rollbackFor：指定遇到特定时强制回滚事务
> - rollbackForClassName：指定遇到特定的多个异常时强制回滚事务。该属性值可以指定多个异常名。
> - timeout：指定事务的超时时间

``` java
public class NewsDaoImpl implements NewsDao
{
	private DataSource ds;
	public void setDs(DataSource ds)
	{
		this.ds = ds;
	}
	@Transactional(propagation=Propagation.REQUIRED ,
		isolation=Isolation.DEFAULT , timeout=5)
	public void insert(String title, String content)
	{
		JdbcTemplate jt = new JdbcTemplate(ds);
		jt.update("insert into news_inf"
			+ " values(null , ? , ?)"
			, title , content);
		// 两次插入的数据违反唯一键约束
		jt.update("insert into news_inf"
			+ " values(null , ? , ?)"
			, title , content);
		// 如果没有事务控制，则第一条记录可以被插入
		// 如果增加事务控制，将发现第一条记录也插不进去。
	}
}
```
还需要在xml中添加如下配置
`<tx:annotation-driven transaction-manager="transactionManager"/>`
