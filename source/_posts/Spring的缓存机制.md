---
title: Spring的缓存机制
categories: 
	- [JAVA后端开发, Spring]
tags: 
	- Spring
	- 学习笔记
date: 2018-12-25 00:00:00
---

# Spring的缓存机制 
Spring的缓存不是一种具体的缓存实现方案，它底层需要依赖EhCache、Guava等具体的缓存工具。应用程序只要面向Spring缓存API编程，应用底层的缓存实现可以在不同的缓存之间自由切换，应用程序无须任何改变，只需要对配置文件略作修改即可。

## 启用Spring缓存
为了启用Spring缓存，需要在配置文件中导入cache:命名空间。
导入cache:命名空间之后，启用Spring缓存还需要两步：
> + 在Spring配置文件中添加`<cache:annotation-driven cache-mangager="缓存管理器ID"/>`,该元素指定Spring根据注解来启用Bean级别或方法的缓存
> + 针对不同的缓存实现配置对应的缓存管理器

### Spring内置缓存实现的配置
Spring内置的缓存实现只是一种内存中的缓存，并非真正的缓存实现，因此通常指能用于简单的测试环境，不建议在实际项目中使用Spring内置的缓存实现。
Spring内置的缓存实现使用SimpleCacheManager作为缓存管理器，使用SimpleCacheManager配置缓存非常简单，直接在Spring容器中配置该Bean，然后通过`<property.../>`驱动该缓存管理器执行setCaches()放来来设置缓存区即可。
SimpleCacheManager是一种内存中的缓存区，底层直接使用了JDK的ConcurrentMap来实现缓存，SimpleCacheManager使用了ConcurrentMapCacheFactoryBean作为缓存区，每个ConcurrentMapCacheFactoryBean配置一个缓存区
``` xml
<!-- 使用SimpleCacheManager配置Spring内置的缓存管理器 -->
<bean id="cacheManager" class="org.springframework.cache.suppot.SimpleCacheManager">
    <!-- 配置缓存区 -->
    <property name="caches">
        <set>
            <!-- 使用ConcurrentMapCacheFactoryBean配置缓存区
                下面列出多个缓存区，p:name用于为缓存区指定名字 -->
            <bean class="org.springframeword.cache.concurrent.ConcurrentMapCacheFactoryBean" p:name="default"/>
            <bean class="org.springframeword.cache.concurrent.ConcurrentMapCacheFactoryBean" p:name="users"/>
        </set>
    </property>
```
上面配置文件使用SimpleCacheManager配置了Spring内置的缓存管理器，并为该缓存管理器配置列两个缓存区：default和users————这些缓存区的名字很重要，后面使用注解驱动缓存时需要根据缓存区的名字来讲缓存数据放入指定缓存区中。

在实际应用中，开发者可以根据自己的需要，配置更多的缓存区，一般来说，应用有多少个组件需要缓存，程序就应该配置多少个缓存区

### EhCache缓存实现的配置
为了使用EhCache，需要在应用的类加载路径下添加一个ehcache.xml配置文件。例如，使用如下echcache.xml文件
``` xml
<?xml version="1.0" encoding="gbk"?>
<ehcache>
    <diskStore path="java.io.tmpdir" />
	<!-- 配置默认的缓存区 -->
    <defaultCache
        maxElementsInMemory="10000"
        eternal="false"
        timeToIdleSeconds="120"
        timeToLiveSeconds="120"
        maxElementsOnDisk="10000000"
        diskExpiryThreadIntervalSeconds="120"
        memoryStoreEvictionPolicy="LRU"/>
	<!-- 配置名为users的缓存区 -->
    <cache name="users"
        maxElementsInMemory="10000"
        eternal="false"
        overflowToDisk="true"
        timeToIdleSeconds="300"
        timeToLiveSeconds="600" />
</ehcache>
```
配置文件解析：
> + diskStore ： ehcache支持内存和磁盘两种存储
>   + path ：指定磁盘存储的位置
> + defaultCache ： 默认的缓存
> + cache ：自定的缓存，当自定的配置不满足实际情况时可以通过自定义（可以包含多个cache节点）
>   + cache ：自定的缓存，当自定的配置不满足实际情况时可以通过自定义（可以包含多个cache节点）
>   + maxElementsInMemory ：内存中允许存储的最大的元素个数，0代表无限个
>   + clearOnFlush：内存数量最大时是否清除。
>   + eternal ：设置缓存中对象是否为永久的，如果是，超时设置将被忽略，对象从不过期。根据存储数据的不同，例如一些静态不变的数据如省市区等可以设置为永不过时
>   + timeToIdleSeconds ： 设置对象在失效前的允许闲置时间（单位：秒）。仅当eternal=false对象不是永久有效时使用，可选属性，默认值是0，也就是可闲置时间无穷大。
>   + timeToLiveSeconds ：缓存数据的生存时间（TTL），也就是一个元素从构建到消亡的最大时间间隔值，这只能在元素不是永久驻留时有效，如果该值是0就意味着元素可以停顿无穷长的时间。
>   + overflowToDisk ：内存不足时，是否启用磁盘缓存。
>   + maxEntriesLocalDisk：当内存中对象数量达到maxElementsInMemory时，Ehcache将会对象写到磁盘中。
>   + maxElementsOnDisk：硬盘最大缓存个数。
>   + diskSpoolBufferSizeMB：这个参数设置DiskStore（磁盘缓存）的缓存区大小。默认是30MB。每个Cache都应该有自己的一个缓冲区。
>   + diskPersistent：是否在VM重启时存储硬盘的缓存数据。默认值是false。
>   + diskExpiryThreadIntervalSeconds：磁盘失效线程运行时间间隔，默认是120秒。

上面的配置文件中同样配置列两个缓存区，其中第一个是用于配置匿名的、默认的缓存区，第二个才是配置了名为users的缓存区。如果需要，完全可以复制多个<cache.../>元素，用于配置多个有名字的缓存区。这些缓存区的名字很重要，后面使用注解驱动缓存时需要根据缓存区的名字来讲缓存数据放入指定缓存区中。

Spring使用EhCacheCacheManager作为EhCache缓存实现的缓存管理器，因此只要该对象配置Spring容器中，它就可以作为缓存管理器使用，但EhCacheCacheManager底层需要依赖一个net.sf.ehcache.CacheManager作为实际的缓存管理器。
为了将net.sf.ehcache.CacheManager纳入Spring容器的管理之下，Spring提供了EhCacheMangagerFactoryBean工厂Bean，该工厂实现了`FactoryBean<CacheManage>`接口。当程序吧EhCacheMangagerFactoryBean部署在Spring容器中，并通过Spring容器请求该工厂Bean时，实际返回的是它的产品————也就是CacheManage对象
``` xml
<beans>
	<context:component-scan 
		base-package="org.crazyit.app.service"/>
		
	<cache:annotation-driven cache-manager="cacheManager" />

	<!-- 配置EhCache的CacheManager
	通过configLocation指定ehcache.xml文件的位置 -->
	<bean id="ehCacheManager"
		class="org.springframework.cache.ehcache.EhCacheManagerFactoryBean"
		p:configLocation="classpath:ehcache.xml"
		p:shared="false" />
	<!-- 配置基于EhCache的缓存管理器
	并将EhCache的CacheManager注入该缓存管理器Bean -->
	<bean id="cacheManager"
		class="org.springframework.cache.ehcache.EhCacheCacheManager"
		p:cacheManager-ref="ehCacheManager" > 
	</bean>
	
</beans>
```

## 使用@Cacheable执行缓存
@Cacheable可用于修饰类或修饰方法，当使用@Cacheable修饰类时，用于告诉Spring在类级别上进行缓存————程序调用该类的实例的任何方法时都需要缓存，而且共享同一个缓存区；当使用@Cacheable修饰方法时，用于告诉Spring在方法级别上进行缓存————只有当程序调用该方法时才需要缓存

### 类级别的缓存
使用@Cacheable修饰类时，就可控制Spring在类级别进行缓存，这样当程序调用该类的任意方法时，只要传入的参数相同，Spring就会使用缓存

``` java
@Service("userService")
// 指定将数据放入users缓存区
@Cacheable(value = "users")
public class UserServiceImpl implements UserService
{
	public User getUsersByNameAndAge(String name, int age)
	{
		System.out.println("--正在执行findUsersByNameAndAge()查询方法--");
		return new User(name, age);
	}
	public User getAnotherUser(String name, int age)
	{
		System.out.println("--正在执行findAnotherUser()查询方法--");
		return new User(name, age);
	}
}
```

当程序第一次调用该类的实例的某个方法时，Spring缓存机制会将该方法返回的数据放入指定缓存区。以后程序调用该类的实例的任何方法时，只要传入的参数相同，Spring将不会真正执行该方法，而是直接利用缓存区中的数据

使用@Cacheable是可指定如下属性：
> + value：**必须属性**。该属性可指定多个缓存区的名字，用于指定将方法返回值放入指定的缓存区内
> + key：通过SpEL表达式显式指定缓存的key
> + condition：该属性指定一个返回boolean值的SpEL表达式，只有当该表达式返回true时，Spring才会缓存方法返回值
> + unless：该属性指定一个返回boolean值的SpEL表达式，当该表达式返回true时，Spring就不缓存方法返回值

与@Cache注解功能类似的还有一个@Cacheput注解，与@Cacheable不同的是，@Cacheput修饰的方法不会读取缓存区中的数据————这以为着不管缓存区是否已有数据，@Cacheput总会告诉Spring要重新执行这些方法，并在此将方法返回值放入缓存区

### 方法级别的缓存
使用@Cacheable修饰方法时，就可控制Spring在方法界别进行缓存，这样当程序调用该方法时，只要传入的参数相同，Spring就会使用缓存
``` java
@Service("userService")
public class UserServiceImpl implements UserService
{
	@Cacheable(value = "users1")
	public User getUsersByNameAndAge(String name, int age)
	{
		System.out.println("--正在执行findUsersByNameAndAge()查询方法--");
		return new User(name, age);
	}
	@Cacheable(value = "users2")
	public User getAnotherUser(String name, int age)
	{
		System.out.println("--正在执行findAnotherUser()查询方法--");
		return new User(name, age);
	}
}
```

上面代码中指定getUsersByNameAndAge()和getAnotherUser()方法分别使用不同的缓存区，这意味着两个方法都会缓存，但由于它们使用了不同的缓存区，因此它们不能共享缓存区数据

## 使用@CacheEvict清除缓存
被@CacheEvict注解修饰的方法可用于清除缓存，使用@CacheEvict注解时可指定如下属性：
> + value：**必需属性**。用于指定该方法用于清除哪个缓存区的数据
> + allEntries：该属性指定是否清空整个缓存区
> + beforeInvocation：该属性指定是否在执行方法之前清除缓存。默认实在fang
> + candion：该属性指定一个SpEL变道时，只有当该表达式为true时才清除缓存
> + key：通过SpEL表达式显式指定缓存的key

``` java
@Service("userService")
@Cacheable(value = "users")
public class UserServiceImpl implements UserService
{
	public User getUsersByNameAndAge(String name, int age)
	{
		System.out.println("--正在执行findUsersByNameAndAge()查询方法--");
		return new User(name, age);
	}
	public User getAnotherUser(String name, int age)
	{
		System.out.println("--正在执行findAnotherUser()查询方法--");
		return new User(name, age);
	}
	// 指定根据name、age参数清除缓存
	@CacheEvict(value = "users")
	public void evictUser(String name, int age)
	{
		System.out.println("--正在清空"+ name
			+ " , " + age + "对应的缓存--");
	}
	// 指定清除user缓存区所有缓存数据
	@CacheEvict(value = "users" , allEntries=true)
	public void evictAll()
	{
		System.out.println("--正在清空整个缓存--");
	}
}
```

``` Java
	public static void main(String[] args)
	{
		ApplicationContext ctx =
			new ClassPathXmlApplicationContext("beans.xml");
		UserService us = ctx.getBean("userService" , UserService.class);
		// 调用us对象的2个带缓存的方法，系统会缓存两个方法返回的数据
		User u1 = us.getUsersByNameAndAge("孙悟空", 500);
		User u2 = us.getAnotherUser("猪八戒", 400);
		//调用evictUser()方法清除缓存中指定的数据
		us.evictUser("猪八戒", 400);
		// 由于前面根据"猪八戒", 400缓存的数据已经被清除了，
		// 因此下面代码会重新执行，方法返回的数据将被再次缓存。
		User u3 = us.getAnotherUser("猪八戒", 400);  // ①
		System.out.println(u2 == u3); // 输出false
		// 由于前面前面已经缓存了参数为"孙悟空", 500的数据，
		// 因此下面代码不会重新执行，直接利用缓存中的数据。
		User u4 = us.getAnotherUser("孙悟空", 500);   // ②
		System.out.println(u1 == u4); // 输出true
		// 清空整个缓存。
		us.evictAll();
		// 由于整个缓存都已经被清空，因此下面两行代码都会重新执行
		User u5 = us.getAnotherUser("孙悟空", 500);
		User u6 = us.getAnotherUser("猪八戒", 400);
		System.out.println(u1 == u5); // 输出false
		System.out.println(u3 == u6); // 输出false
	}
```
