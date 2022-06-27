---
title: 深入使用 Spring
tags: 
    - Spring
    - 学习笔记
categories:  
    - [JAVA后端开发, Spring]
date: 2018-12-25 00:00:00
---
# 深入使用 Spring
## 两种后处理器

> + Bean后处理器：这种后处理器会对容器中的 Bean进行后处理，对 Bean 进行额外加强
> + 容器后处理器：这种后处理器对 IoC 容器进行后处理，用于增强容器功能

### Bean 后处理器
Bean 后处理器是一种特殊的 Bean，这种特殊的 Bean 并不对外提供服务，它甚至无需 id 属性，它主要负责对容器中其他 Bean 执行后处理，例如为容器中的目标 Bean 生成代理等，这种Bean 被称作**Bean后处理器**

Bean后处理器会在 Bean 实例创建成功之后，对 Bean 实例进行进一步的增强处理，Bean 后处理器必须实现 BeanPostProcessor 接口，该接口包含如下两个方法：

``` java
// 该方法的第一个参数是系统即将进行后处理的 Bean 实例，第二个参数是该 Bean 的配置 id
Object postProcessAfterInitialization(Object bean, String beanName)throws BeansException
// 该方法的第一个参数是系统即将进行后处理的 Bean 实例，第二个参数是该 Bean 的配置 id
Object postProcessAfterInitialization(Object bean, String beanName)throws BeansException
```
实现 Bean 后处理器必须实现这两个方法，这两个方法会对容器的 Bean 进行后处理，会在目标 Bean 初始化之前、初始化之后分别被回调，这两个方法用于对容器中的 Bean 进行增强处理。

实现 BeanPostProcessor 接口的 Bean 可对 Bean 进行任何操作，包活完全忽略这次回调。BeanPostProcessor 通常用来检查标记接口，或者做如将 Bean 包装成一个 Proxy 的事情，Spring的很多工具类就是通过 Bean 后处理器完成的。  

在配置文件中配置 Bean 后处理器和配置普通Bean 一样，通常程序无须主动获取 Bean 后处理器，因此配置文件可以无需为 Bean 后处理器指定 id 属性。

如果使用 BeanFactory 作为 Spring 容器，则必须手动注册 Bean 后处理器，程序必须获取 Bean 后处理器实例，然后手动注册。在这种需求下，程序可能需要在配置文件中为 Bean 处理器指定 id属性，这样才能让 Spring 容器先获取 Bean 后处理器，然后注册它。
``` java
public class BeanTest
{
    public static void mian(String[] args)
    {
        Resource isr = new ClassPathResource("beans.xml");
        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
        new XMLBeanDefinitionReader(beanFactor).loadBeanDefinitions(isr);
        BeanPostProcessor bp = (BeanPostProcessor)beanFactor.getBean("bp");
        beanFactory.addBeanPostProcessor(bp);
    }
}
```

下面是 Spring 常用的两个后处理器：
> + BeanNameAutoProxyCreator：根据 Bean 实例的 name 属性，创建 Bean 实例的代理
> + DefaultAdvisorAutoProxyCreator：根据提供的 Advisor，对容器中的所有 Bean 实例创建代理

如果需要对容器中的某一批 Bean 进行通用的增强处理，则可以考虑使用 Bean 后处理器

### 容器后处理器
容器后处理器负责处理容器自身，必须实现 BeanFactoryPostProcessor 接口，该接口需实现如下方法：
`postProcessBeanFactory(ConfigurableListBeanFactory beanFactory)`
该方法知识对 Spring 容器进行后处理，方法无须人和返回值。

类似于 BeanPostProcessor，ApplicationContex 可自动检测到容器中的容器后处理器，并且自动注册容器后处理。但若使用 BeanFactory 作为Spring容器，则必须通过手动调用该容器后处理器来处理 BeanFactory 容器。
Spring 已提供如下几个常用的容器后处理器：
> - PropertyPlaceholderConfigurer：属性占位符配置器
> - PropertyOverrideConfigurer：重写占位符配置器
> - CustomAutowireConfigurer：自定义自动装配的配置器
> - CustomScopeConfigurer：自定义作用域的配置器  

如果有需要，程序可以配置多个容器后处理器，多个容器后处理器可设置 order 属性来控制容器后处理器的执行顺序。为了给容器后处理器指定 order 属性，则要求容器后处理器必须实现 Ordered接口。

容器后处理的作用域范围是容器级，它只对容器本身进行处理，并且总是在容器实例化人和其他的 Bean 之前，读取配置文件的元数据，并有可能修改这些元数据。

#### 属性占位符配置器
PropertyPlaceholderConfigurer，它是一个容器后处理器，负责读取 Properties 属性文件里的属性值，并将这些属性值设置成 Spring 配置文件的数据，如下所示：
``` xml
<bean class="org.springframework.bean.factory.config.PropertyPlaceholderConfigurer">
    <property name="locations">
        <list>
            <value>dbconn.properties</value>
            <!-- 如果有多个属性配置文件，可依次在此列出 -->
        </list>
    </property>
</bean>
<!-- 定义数据源 -->
<bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource" 
destort-method="close" p:driverClass="${jdbc.driverClassName}"
p:jdbcUrl="${jdbc.url}" p:user="${jdbc.username}" p:password="${jdbc.password}"/>
```

dbconn.properties 配置文件内容如下：
```
jdbc.driveClassName=com.mysql.cj.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/mystudy
jdbc.username=root
jdbc.password=root
```

PropertyPlaceholderConfigurer就是通过"${}"占位符将配置文件中的内容配置给 Bean

如果采用基于 XML Schema 的配置文件而言，如果导入了 context:命名空间，则可才用如下方式：
`<context:property-placeholder location="classpath:db.properties"/>`

#### 重写占位符配置器
PropertyOverrideConfigurer的属性文件指定的信息可以直接覆盖 Spring 配置文件中的元数据。当 XML 配置文件和属性文件指定的元数据不一致时，属性文件的信息取胜。
使用PropertyOverrideConfigurer的属性文件，每条属性应该保持如下格式：
`beanId.property=value`
beanId 是属性占位符试图覆盖的 Bena 的 id，property 是试图覆盖的属性名(对应于调用 setter 方法)
``` xml
<bean class="org.springframework.bean.factory.config.PropertyOverrideConfigurer">
    <property name="locations">
        <list>
            <value>dbconn.properties</value>
            <!-- 如果有多个属性配置文件，可依次在此列出 -->
        </list>
    </property>
</bean>
<!-- 定义数据源，配置该 Bean 时没有指定人和信息，但 Properties文件里的信息将会覆盖该 
    Bean 的属性值 -->
<bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource" 
destort-method="close" p:driverClass="${jdbc.driverClassName}"
p:jdbcUrl="${jdbc.url}" p:user="${jdbc.username}" p:password="${jdbc.password}"/>
```
dbconn.properties配置文件如下所示：
```
dataSource.driveClassName=com.mysql.cj.jdbc.Driver
dataSource.url=jdbc:mysql://localhost:3306/mystudy
dataSource.username=root
dataSource.password=root
```
此配置文件中的dataSource对应 beanid，如不存在则发生错误

如果采用基于 XML Schema 的配置文件而言，如果导入了 context:命名空间，则可才用如下方式：
`<context:property-override location="classpath:db.properties"/>`

## Spring 的自动装配 
### 搜索 Bean
如果希望 Java 类可以被当做 Bean 处理需使用如下注解修饰：
> + @Component：标注一个普通的 Spring Bean类
> + @Controler：标注一个控制器组件类
> + @Service：标注一个业务逻辑组件类
> + @Repository：标注一个 DAO 组件类

通过以上注解配置的 Bean ，默认的 Bean Id 为类名（将第一个字母小写），如果需要为 Bean 指定 Id，则需要在注解内手动配置，例如`@Component("mybean")`。也可使用 @Named注解配置 Bean

需要在配置文件中指定Bean搜索搜索路径，Spring 将会自动搜索该路径下的所有 Java 类，并根据这些 Java 类来创建 Bean 实例。

如果使用 XML 配置方式，需增加`<context:component-scan base-package="com.learn.app"/>`开启自动扫描。此处可以添加`<include-filter.../>`子元素用于强制 Spring处理这些类，即使这些类没有使用注解修饰；添加`<exclude-filter.../>`子元素用于强制将某些 Spring 注解修饰的类排除在外。使用这两个子元素时，需要指定如下两个属性：
> + type: 指定过滤器类型，支持如下4种过滤器
>   > + annotation：注解过滤器，该过滤器需要指定一个注解名
>   > + assignable：类名过滤器，该过滤器直接指定一个 Java 类
>   > + regex：正则表达式过滤器，匹配该正则表达式的 Java 类将满足过滤规则
>   > + aspectj：AspectJ 过滤器
> 
> + expression：指定过滤器所需要的表达式

如果通过 Java 配置方式，需要在 Java 配置类上增加`@ComponentScan`注解

Spring提供了 @Lookup 注解来执行 lookup 方法注入，使用@Lookup直接修饰需要执行 lookup 方法注入的方法，通过指定 value 属性指定Spring实现该方法的返回值

### 指定 Bean 的作用域
可使用`@Scope`注解修饰，只要在该注解中提供作用域的名称即可。

@Scope提供如下三个属性：
> + proxyMode：指定是否应将组件配置为作用域代理，如果是，则指定代理应基于接口还是基于子类。支持如下属性：
> 
>   > + ScopedProxyMode.DEFAULT：它通常表示除非在组件扫描指令级别配置了不同的默认值，否则不应创建范围代理
>   > + ScopedProxyMode.INTERFACES：创建一个JDK动态代理，实现目标对象的类所公开的所有接口
>   > + ScopedProxyMode.NO：不要创建作用域代理
>   > + ScopedProxyMode.TARGET_CLASS：创建基于类的代理（使用CGLIB）
> + scopeName：为组件/Bean 提供作用域名称，默认为空字符串（“”），表示SCOPE_SINGLETON。
> + value：scopeName的别名

Spring4.3中新增了@ApplicationScope、@SessionScope、@RequestScope，它们分别对应@Scope("application")、@Scope("session")、@Scope("request"),且它们的 proxyMode 属性被设置为 ScopeProxyMode.TARGET_CLASS。

在 XML 中配置需在`bean`中指定 scope 属性， 使用`<aop:scope-proxy>`指定 proxyMode 属性

如果需要使用自定义作用域解析策略，则在配置扫描器是指定解析器的全限定类名即可
`<context:component-scan base-package="com.learn.app" scope-resolver="org.learn.app.MyScopeResolver"/>`

### 使用@Resource 和@Value 配置依赖
使用@Resource 与 `<property.../>`元素的 ref 属性有相同的效果，name 属性指定所引用的 Bean id

使用@Value 与`<property.../>`元素的  value 属性有相同的效果，还可以使用 SpEL 表达式

@Resource、@Value不仅可以修饰 setter 方法，也可以直接修饰实例变量。

Spring 允许使用@Resource 时省略 name 属性，当使用省略 name 属性的@Resource 修饰的 setter 方法时，name 属性默认为该 setter 方法去掉前面的 set 子串、首字母小写后得到的子串；当使用省略name 属性的@Resource 修饰实例变量时，name 属性值默认与该实例变量同名。

### 使用@PostConstruct 和@PreDestory 定制声明周期
@PostConstruct 作用与在`<bean.../>`中指定 init-method 属性相同，指定 Bean 的初始化方法

@PreDestory 作用与在`<bean.../>`中指定 destory-method 属性相同，指定 Bean 的销毁方法

它们都用于修饰方法，无需任何属性

### 使用@DependsOn 和@Lazy 改变初始化行为
@DependsOn 用于强制初始化依赖的 Bean；@Lazy 用于指定该 Bean 是否取消预初始化

@DependsOn可以修饰 Bean 类或方法，使用该注解时可以指定一个字符串数组作为参数，每个数组元素对应于一个强制初始化的 Bean

@Lazy 修饰 Spring Bean 类是否用于指定该 Bean 的预初始化行为，使用该注解时刻指定一个 boolean 型的 value 属性，该属性决定是否要预初始化该 Bean

### 自动装配与精确装配
Spring 提供了@Autowired 注解来指定自动装配，@Autowired可以修饰 setter 方法、普通方法、实例变量和构造器。当使用@Autowired标注 setter 方法时，默认采用 byType 自动装配策略

@Autowired修饰 setter 方法时，通过该 setter 方法的参数类型在 Spring 容器中搜索，如果找到该类型的 Bean，则使用该 Bean 调用 setter 方法进行装配；如果找到多个此类型的 Bean，则Spring 引发异常；如果没找到该类型的 Bean，则什么也不做

@Autowired修饰带多个参数的普通方法时，Spring 会自动到容器中寻找类型匹配的 Bean，如果恰好为每个参数都找到一个类型匹配的 Bean，Spring 会自动以这些 Bean 作为参数来调用方法。

@Autowired修饰 修饰一个实例变量时，Spring 将会把容器中与该实例变量类型匹配的 Bean 设置为该实例变量的值。如果找到多个该类型的 Bean，则 Spring 容器会抛出 BeanCreateException

@Autowired修饰数组类型、集合类型的成员变量和方法时，Spring 会自动搜索容器中所有的该类型实例，并注入 到数组或集合之中。如果程序没有使用泛型来指明元素的类型，则 Spring 将会不知所措。

Spring 提供@Primary 注解，该注解用于将指定的候选 Bean 设置为主候选者 Bean。在 Spring 通过类型选择依赖注入时，会直接匹配该注解修饰的 Bean，无视其他同类型的 Bean。

Spring 提供了@Qualifier 注解，通过该注解允许根据 Bean 的 id 来执行自动装配。可用于修饰实例变量、方法形参
> 注：使用@Autowire 和@Qualifier 实现自动装配，不如直接使用 @Resource 注入依赖

### Spring5新增的注解
在使用@Autowired 注解执行自动装配时，该注解可以指定一个 required属性，该属性默认为 true，这意味着该注解修饰的 Field 或setter 方法必须被依赖注入，否则 Spring 会在初始化容器时报错
> 注：@Autowired与在 xml 中指定 autowired="byType"的自动装配存在区别。使用 xml 中的自动装配配置时，如果找不到自动装配的候选 Bean，Spring 不执行注入也不报错；但使用注解机型自动装配时，找不到会直接报错

Spring5一如如下新注解：
> - @Nullable：该注解主要用于修饰参数、返回值和 Field，声明他们可以为 null
> - @NonNull：该注解主要用于修饰参数、返回值和 Field，声明他们不允许为 null
> - @NonNullApi：该注解用于修饰包，表明该包内 API 的参数、返回值都不应该为 null。如果希望该包内某些参数、返回值可以为 null，则需要使用@Nullable 修饰它们。
> - @NonNullField：该注解用于修饰包，表示该包内的 Field 都不应该是 null。如果希望该包内的某些 Field 可以为 null，则需要使用@Nullable 修饰他们
>

### 使用@Required 检查注入
@Required可用来修饰 setter 方法，用来检查注入，Spring 在创建容器时就执行检查该 setter 方法，如果开发者既没有显式通过`<property.../>`配置依赖，也没有使用自动装配执行依赖注入，Spring 容器会引发 BeanInitializationException 异常

## 环境与 profile
要使用 profile，首先要将所有不同的 Bean 整理到一个或多个 profile 之中，在将应用部署到每个环境时，要确保对应的 profile 处于激活状态。可使用**@Profile**注解指定某个 bean 属于哪一个 profile。**@Profile**可修饰类和方法。只有当规定的profile 被激活时，相应的 Bean （被@Profile 修饰的）才会被创建。没有指定 profile 的 Bean 始终都会被创建，与激活哪一个 profile 无关。

也可通过在 XML 中配置profile，可应用于单个配置文件，也可嵌套使用。
- 可将每一个 profile 配置一个 XML文件，通过在`<beans.../>`中通过 profile 属性配置并指定相应的 profile 名字。在主配置文件中通过`<import.../>`子属性导入profile 配置文件

    ``` xml
    <beans profile="dev">
        <bean id="..." class="..."/>
        <bean id="..." class="..."/>
    </beans>
    ```
    
    ```
    <beans>
        <import resource="dev-profile.xml"/>
        <import resource="qa-profile.xml"/>
        <import resource="prod-profile.xml"/>
    </beans>
    ```
    
- 可将所有的 profile 集合在一个XML配置文件中，通过`<beans.../>`嵌套

    ``` xml
    <beans >
        <beans profile="dev">
            <bean id="..." class="..."/>
            <bean id="..." class="..."/>
        </beans>
        <beans profile="qa">
            <bean id="..." class="..."/>
            <bean id="..." class="..."/>
        </beans>
        <beans profile="prod">
            <bean id="..." class="..."/>
            <bean id="..." class="..."/>
        </beans>
    </beans>
    ```
    
Spring在确定哪个 profile 处于激活状态时，通过依赖两个独立的属性：spring.profiles.active 和 spring.profiles.defaule。如果设置了spring.profiles.active，则使用其所指定的 profile；如果没有设置spring.profiles.active，则使用spring.profiles.defaule指定的 profile；如果两者都没有指定，则没有激活的 profile,只创建没有配置 profile 的 Bean。可通过如下方式来设置这两个属性：
> - 作为 DispatcherServlet 的初始化参数
> - 作为 Web 应用的上下文参数
> - 作为 JNDI 条目
> - 作为环境变量
> - 作为 JVM 系统属性
> - 在集成测试类上，使用@ActiveProfiles 注解配置

## 条件化的 Bean
通过**@Conditional**注解修饰带有@Bean修饰的方法上，如果给定的条件计算为 true，就会创建 bean，否则这个 bean 会被忽略。此注解需要value 属性指定一个到多个`Class<? extends Condition>`作为参数指定条件。

``` java
@Bean
@Conditional(MyCondionalImpl.class)
public TestBean testBean()
{
    return new TestBean();
}
```

Conditon 是一个接口，指定`matches(ConditionContext context, AnnotatedTypeMetadata metadata)`方法用于判断条件。如果满足条件，matches返回 true 则创建 Bean；如果返回 false 则忽略该 Bean。参数类型ConditionContext、AnnotatedTypeMetadata都是接口，可通过该接口方法获取如下信息：
> + ConditionContext接口
> 
>   > + 通过 getRegistry()返回的 BeanDefinitionRegistry 检查 Bean 定义
>   > + 通过 getBeanFactory()返回的 ConfigurableListableBeanFactory 检查 Bean 是否存在，甚至探测 Bean 的属性
>   > + 通过 getEnvirment()返回的 Envirment 检查环境变量是否存在以及它的值是什么
>   > + 通过 getResources()返回 ResourcesLoader所加载的资源
>   > + 通过 getClassLoader()返回的 ClassLoader 加载并检查类是否存在
> 
> + AnnotatedTypeMetadata接口：可用于获取 带有@Bean 注解的方法上还有什么其他注解
> 

``` Java
public MyCondionalImpl implements Condition
{
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata)
    {
        Envirments env = context.getEnvirment();
        return env.containsProperty("mytest");
    }
}
```

上面记录的@Profile 属性就是结合了@Condional的复合注解

## 运行时值注入
可通过 XML 和注解两种方式加载Properties 配置文件到 Spring Envirment 中
> + 通过 XML 配置：
>   `<context:property-placeholder location="classpath:db.properties"/>`
> + 通过注解引入:
>   使用@PropertySource修饰 Java配置类
> 
>   ``` java
>   @Configuration
>   @PropertySource("classpath:/app.properties")
>   public class MyConfig
>   {
>       @Autowired
>       Environment env;
>       public testBean()
>       {
>           return new TestBean(env.getPropery("name"),env.getProperty("phone"));
>       }
>   }
>   ```

### 深入学习 Spring 的 Envirment

通过使用Envirment接口可以通过getProperty()获取到配置信息属性值，其有如下几种重载方式：
> + `String getProperty(String)`：通过字符串查找属性值，找不到返回null
> + `String getProperty(String key,String defaultValue)`：通过字符串查找属性值，找不到返回defaultValue所指定字符串
> + `T getProperty(String key, Class<T> type);`：通过字符串查找属性值，返回type所指定的类型，找不到返回null
> + `T getProperty(String key, Class<T> type, T defaultValue)`:通过字符串查找属性值，返回type所指定的类型，找不到返回defaultValue所指定的值

其他常用方法：
> + `String getRequiredProperty(String key)`：返回与给key关联的属性值（永不为null）。如果请求的属性没找到，则引发IllegalStateException异常
> + `String[] getActiveProfiles()`：返回激活profile名称的数组
> + `String[] getDefaultProfiles()`：返回默认profile名称的数组
> + `boolean acceptsProfiles(String... profiles)`：若果envirment支持给定的profile的话就返回true
> 

可以通过属性占位符来获取配置信息，通过配置 PropertySourcesPlaceholderConfigurer（容器后处理器中介绍）

## 资源访问
Spring 改进了 Java 资源访问的策略，提供了一个 Resource 接口，该接口提供了更强力的资源访问能力，Spring 框架自身大量使用了Resource 来访问底层资源。Resource 接口主要提供如下几个方法：
> + `getInputStream()`：定位并打开资源，返回资源对应的输入流。每次调用都返回新的输入流。调用者负责关闭输入流
> + `exists()`：返回 Resource 所指向的资源是否存在
> + `isOpen()`：返回该资源是否打开，如果资源文件不能多次读取，每次读取结束时应该显式关闭，以防止资源泄露
> + `getDescription`：返回资源的描述信息，用于资源处理出错时输出该信息，通常是全限定文件名或实际URL
> + `getFile`：返回资源对应的 File 对象
> + `getURL`：返回资源对应的 URL 对象
> 

Resource 接口的具体实现类有如下几种：
> + UrlResource：访问网络资源的实现类
> + ClassPathResource：访问类加载路径里资源的实现类
> + FileSystemResource：访问文件系统里资源的实现类
> + ServletContextResource：访问相对于 ServletContext 路径下的资源的实现类
> + InputStreamResource：访问输入流资源的实现类
> + ByteArrayResource：访问字节数组资源的实现类

当执行 Spring 的某个方法时，该方法接受一个代表资源路径的字符串参数，当 Spring识别该字符串参数中包含前缀时，系统将会根据前缀自动选择实现类。

### ResourceLoader 接口和 ResourceLoaderAware 接口
ResourceLoader：该接口的实现类可以获得一个 Resource 实例，包含如下方法：
> `Resource getResource(Sting location)`： 该方法根据位置信息返回一个 Resource 实例。ApplicationContext 的实现类都实现了 ResourceLoader 接口，可用于直接获取 Resource实例。

当 Spring 应用需要进行资源访问时，实际上并不需要直接使用 Resource 实现类，而是调用 ResourceLoader 实例的 getResource()方法来获得资源，ResourceLoader 将会负责选择 Resource 的实现类，也就是确认具体的资源访问策略，从而将应用程序和具体的资源访问策略分离出来。这就是典型的策略模式。

以下是常用的前缀及对应的访问策略
> classpath: ：以ClassPathResource 实例访问类加载路径下的资源
> file: ：以 FileSystemResource 实例访问本地文件系统的资源
> http: : 以 UrlResource 实例访问基于 HTTP 协议的网络资源
> 无前缀：由 ApplicationContext 的实现类决定访问路径

ResourceLoaderAware：该接口的实现类的实例将获得一个 ResourceLoader 的引用，该接口提供 setResourceLoader()方法。类似于 BeanFactroyAware、BeanNameAware 接口，该方法由 Spring 容器负责调用。如果把实现 ResourceLoaderAware 接口的 Bean 部署在 Spring 容器中，Spring 容器会将自身当成 ResourceLoader 作为 setResourceLoader()方法的参数传入。

### 使用 Resource 作为参数
如果 Bean 实例需要访问资源，则有如下两种解决方案：
> + 在代码中获取 Resource 实例。
>   在这种方式下，当程序获取 Resource 实例时，总需要提供 Resource 所在的位置，资源所在的位置将被耦合到代码中，如果资源位置发生改变，则必须改写程序。
> + 使用依赖注入（**推荐使用**）

### 在 ApplicationContext 中使用资源
1. 使用ApplicationContext 实现类指定访问策略
    - ClassPathXmlApplicationContext：对应使用 ClassPathResource 进行资源访问
    - FileSystemXmlApplicationContext：对应使用 FileSystemResource 进行资源访问
    - XmlWebApplicationContext：对应使用 ServletContextResource 进行资源访问
2. 使用前缀指定访问策略<br>
    `ApplicationContext ctx = new FileSystemXmlApplicationContext("classpath:beans.xml")`
    这里虽然使用 FileSystemXmlApplicationContext实现类，但程序依然从累加载路径搜索 beans.xml。相应的还可以使用 http:、ftp:等前缀来确定对应的资源访问策略
    通过前缀指定资源访问策略仅仅对当次访问有效，程序后面进行资源访问时，还是会根据 ApplicationContext 的实现类来选择对应的资源访问策略。
3. classpath\*:的用法
    当使用classpath\*:前缀来指定 XML 配置文件时，系统将搜索类加载路径，找出所有与文件名匹配的文件，分别加载文件中的配置定义，最后合并成一个 ApplicationContext。<br>
    `ApplicationContext ctx = new FileSystemXmlApplicationContext("classpath*:beans*.xml")`
    此处 Spring 会找出类加载路径下所有名称中包含“bean”的xml 文件，合并成一个ApplicationContext
    classpath\*:前缀只对ApplicationContext有效，不可用于Resource
4. file:前缀的用法
    FileSystemApplicationContext会让所有绑定的FileSystemResource实例把绝对路径当成相对路径处理，而不管是否以斜杠开头。建议强制使用file:前缀来区分绝对路径和相对路径


