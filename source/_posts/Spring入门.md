---
title: Spring入门
categories: 
  - [JAVA后端开发, Spring]
tags: 
	- Spring
	- 学习笔记
date: 2018-12-25 00:00:00
---

# Spring入门
## 使用Spring容器
Spring有两个核心接口，BeanFactory和ApplicationContext,其中ApplicationContext是BeanFactory的子接口。它们都可代表Spring容器，Spring容器是生成Bean实例的工厂，并管理容器中的Bean.
### Spring容器
BeanFactory是Spring最基本的容器，包含如下几个基本方法：
``` java
// 判断Spring容器中是否包含id为name的Bean实例
boolean containsBean(String name);
// 获取Spring容器中输入requiredType类型的、唯一的Bean实例
<T> T getBean(Class<T> requiredType);
// 返回容器id为name的Bean实例
Object getBean(String name);
// 返回容器中id为name，并且类型类requiredType的Bean
<T> T getBean(String name,class<T> requiredType);
// 返回容器中id为name的Bean实例的类型
```
BeanFactory常用的实现类为DefaultListableBeanFactory
``` java
// 搜索类加载路径下的beans.xml文件创建Resource对象
Resource isr = new ClassPathResource("beans.xml");
DefaultListableBeanFactory beanFactory =  new DefaultListableBeanFactory;
new XmlBeanDefinitionReader(beanFactor).loadBeanDefinitions(isr);
```
如果应用需要加载多个配置文件来创建Spring容器，则应该采用BeanFactory的子接口ApplicationContext来创建BeanFactory的实例。ApplicationContext接口包含FileSystemXmlApplicationContext和ClassXmlApplicationContext两个常用类，分别代表从类路径加载配置文件和从文件系统加载配置文件。
``` java
ApplicationContext appContext = new ClassPathXmlApplicationContext("beans.xml", "server.xml");
```
### 使用ApplicationContext
ApplicationContext支持BeanFactory的全部功能，还支持如下额外功能：
> - ApplicationContext默认会初始化所有的singleton Bean,也可以通过配置取消初始化
> - ApplicationContext继承MessageSource接口，提供国际化支持
> - 同时加载多份配置文件
> - 事件机制
> - 以声明方式启动并创建Spring容器  
> 

当系统创建ApplicationContext容器时，默认会预初始化所有的singleton Bean，所以系统前期创建ApplicationContenxt将消耗较大的系统开销，但一旦初始化完成，程序后面获取singleton Bean实例会有较好的性能。  

BeanFactory在创建时并不会初始化所有的Bean，只有在getBean方法显式调用时才会初始化Bean。  

ApplicationContext包括BeanFactory的全部功能，因此建议优先使用ApplicationContext。除非对于哪些内存非常关键的引用，才考虑使用BeanFactory.  

无可阻止Spring容器预初始化容器中的singleton Bean，可以为`<bean.../>`元素指定lazy-init="true",该属性用于阻止预初始化该Bean

### ApplicationContext的事件支持
ApplicationContext的事件机制是**观察者模式**的实现。如果容器中有一个ApplicationListener Bean,每当ApplicationContext发布ApplicationEvent时，ApplicationListeners Bean将自动触发

Spring的事件框架依赖如下两个重要成员：
> - ApplicationEvent：容器事件，必须由ApplicationContext发布
> - ApplicationListener：监听器，可由容器中的任何监听器Bean担任  
> 	
>
> 只要一个Java类继承了ApplicationEvent基类，那该对象就可以作为Spring容器的容器事件。

容器事件的监听器类必须实现ApplicationListener接口，该接口必须实现如下方法：
> - onApplicationEvent(ApplicationEvent event)：每当容器内发生任何事件时，该方法自动被触发
> 
在配置文件配置监听类Bean时，可省略id属性，Spring容器会自动识别该监听器  

当系统创建Spring容器、加载Spring容器时会触发容器事件，容器事件监听器可以监听到这些事件。程序也可调用ApplicaionContext的publicEvent()方法来主动触发容器事件。
如果开发者需要在Spring容器初始化、销毁时回调自定义方法，就可以通过事件监听器来实现。
> 注：如果希望Bean发布容器事件，则必须让该Bean实现ApplicationContextAware或BeanFactoryAware接口，以获得对容器的引用。
> 
### 让Bean获取Spring容器
为了让Spring获取它所在的Spring容器，可以让Bean实现BeanFactoryAware接口，实现如下方法：
``` java
setBeanFactory(BeanFacotry beanFactory); //方法中的参数指向创建它的BeanFactory
```
类似的还有ApplicationContextAware接口，需实现的方法如下：
``` java
setApplicationContext(ApplicationContext context); //方法中的参数指向创建它的ApplicationContext
```
Spring容器会检查所有的Bean，如果发现某个Bean实现了如上接口，将自己调用该Bean的setBeanFactory或setApplicaionContext方法将自身作为参数传递给该方法。

## 容器中的Bean

### Bean的基本定义和Bean别名
`<beans.../>`元素是Spring配置文件的根元素，可以指定如下属性
> - default-lazy-init：指定该\<beans.../\>元素下配置的所有Bean默认的延迟初始化行为
> - default-merge：指定该\<beans.../\>元素下配置的所有Bean的merge行为
> - default-autowire：指定该\<beans.../\>元素下配置的所有Bean默认的自动装配行为
> - default-autowire-candidates：指定该\<beans.../\>元素下配置的所有Bean默认是否作为自动自动装配的候选Bean
> - default-init-method：指定该\<beans.../\>元素下配置的所有Bean默认的初始化方法
> - default-destory-method：指定该\<beans.../\>元素下配置的所有Bean默认的回收方法

在单个Bean中应用，只需将defalut去掉便可以应用于该Bean，\<bean.../\>下指定的属性会覆盖\<beans.../\>下指定的属性。

在定义Bean时，通常需指定如下两个属性：
> - id：确定该Bean在Spring容器中的唯一标识，容器对Bean的管理、访问，以及该Bean的依赖关系都依赖于此属性
> - class：指定该Bean的具体实现类，不能是接口。
> 
id属性默认不能包含特殊字符，如\*@/等。可通过name属性指定别名使用
> - name：该属性指定一个Bean示例的标识名，表明将为该Bean示例指定别名
> - alias：指定一个别名
> 

``` xml
<bean id="person" class="..." name="#abc,@123,abc*"/>
<alias name="person" alias="jack"/>
<alias name="jack" alias="jackee"/>
```
### 容器中Bean的作用域
Spring支持如下6种作用域：
> + singleton：单例模式，在整个SpringIoC容器中，该实例只生成一次。默认属性
> + prototype：每次通过同期的getBean()方法获取prototype作用域的Bean时，都会生成新的实例。
> + request：同一次请求中获取的Bean只生成一次
> + session：同一个会话中获取的Bean只生成一次
> + application：对于整个Web应用中获取的Bean为相同的实例
> + websocket：在整个WebSocket的通信过程中，只生成一个实例

常用的为singleton和pototype作用域，其它四种作用域只应用于Web环境中。  
对于singleton作用域实例，由容器负责跟踪Bean实例的状态，负责该Bean的整个生命行为。对于prototype作用域的的Bean,容器只负责生成，一旦创建成功便不再跟踪

### 配置依赖
> + 设值注入：通过`<property.../>`元素驱动Spring执行setter方法
> + 构造注入：通过`<constructor-arg.../>`元素驱动Spring执行带参数的构造器
> 
通常不建议使用配置文件管理Bean的基本类型的属性值；通常只使用配置文件管理容器中Bean之间的依赖关系。  
通过`<constructor-arg.../>`配置依赖时时根据参数顺序进行注入依赖，也可通过index属性显式指定参数顺序：

``` xml
<bean id="Person" class="com.learn.Person">
  <constructor-arg index="1" value="35"/>
  <constructor-arg index="0" value="张三"/>
</bean>
```

### 设置普通属性值
`<value.../>`元素用于指定基本类型及其包装、字符串类型的参数值，Spring通过XML解析器来解析出这些数据，然后利用java.beans.PropertyEditor完成类型转换：从String类型转换为所需的参数类型

### 配置合作者Bean
`<ref.../>`元素用于指定所依赖的其它Bean，指定该Bean的id属性值

### 使用自动装配注入合作者Bean
autowire、default-autowire接收如下参数值：
> + no：不使用自动装配。Bean依赖必须通过ref显式配置。**默认配置，不建议更改**
> + byName：根据setter方法名进行自动装配。Spring容器查找容器中的全部Bean，找出其id与setter方法名去掉set前缀，并小写首字母后同名的Bean来完成注入。如没有找到，则不会进行任何注入。
> + byType：根据setter方法的形参类型来自动匹配。Spring容器查找容器中的全部Bean，如果正好有一个Bean类型与setter方法的形参类型匹配，就自动注入这个Bean；如果找到多个这样的Bean，则抛出异常；如果没有找到，则什么都不会发生。
> + constructor：与byType类型，区别是用于自动匹配构造器的参数
> + autodetect：Spring容器根据Bean内部结构，自行决定使用constructor或byType策略。如果找到一个默认的构造参数，那么就会应用byType策略。
> 
> 当一个Bean既使用自动装配依赖，又使用ref显式指定依赖时，则显式指定的依赖覆盖自动装配的依赖。 
> 
如果希望将某些Bean排除在自动装配之外，不作为Spring自动装配策略的候选者，可`<bean.../>`设置该Bean的autowire-candidate属性为false。还可以通过在`<beans.../>`中指定default-autowire-candidates属性将一批Bean排除在自动装配之外（该属性值允许使用模式字符串，可指定多个模式字符串）。  

自动装配将Bean之间的耦合关系降低到代码层次，不利于高层次解耦；降低了依赖关系的透明性和清晰性。故在大型项目中不推荐使用自动装配。

### 注入嵌套Bean
如果某个Bean所依赖的Bean不想被Spring容器直接访问，则可以使用嵌套Bean。
把`<beans.../>`配置成`<property.../>`或`<constructor-ages.../>`的子元素，那么该`<beans.../>`元素配置的Bean仅仅作为setter注入、构造注入的参数。由于容器不能获取嵌套Bean，因此可不需指定id属性。

``` xml
<bean id="chinese" class="org.learn.impl.Chinese">
  <property name="axe">
    <bean class="org.learn.impl.SteelAxe"/>
  </property>
</bean>
```

### 注入集合值
如果需要使用形参为集合的setter方法，或调用形参为集合的构造器，则可使用集合元素\<list.../\>、\<set.../\>、\<map.../\>、\<props.../\>分别来设置类型为List、Set、Map和Properties的集合参数值。

``` xml
<bean id="chinese" class="org.learn.impl.Chinese">
  <property name="schools">
    <list>
      <value>小学</value>
      <value>中学</value>
      <value>大学</value>
    </list>
  </property>
  <property name="scores">
    <map>
      <entry key="数学" value="87"/>
      <entry key="英语" value="90"/>
    </map>
  </property>
  <property name="phaseAxes">
    <map>
      <entry key="原始社会" value-ref="stoneAxe"/>
      <entry key="农业社会" value="steelAxe"/>
    </map>
  </property>
  <property name="health">
    <props>
      <prop key="血压"/>正常</prop>
      <prop key="身高"/>175</prop>
    </props>
  </property>
  <property name="axes">
    <set>
      <value>字符串</value>
      <bean class="..." />
      <ref bean="stoneAxe"/>
    </set>
  </property>
</bean>
```

`<list.../>`、`<set.../>`、`<key../>`可接受如下子元素：
> + value：指定集合元素是基本数据类型或字符串类型值
> + ref：指定集合元素是容器中的另一个Bean示例
> + bean：指定集合元素是一个嵌套Bean
> + list、set、map及props：指定集合元素又是集合

`<props.../>`元素用于配置Properties类型的参数值，其key和value只能是字符串

`<map.../>`元素的`<entry.../>`子元素配置一组键值对，其支持如下属性:
> + key：如果Map key是基本类型或字符串
> + key-ref：如果Map key是容器中的另外一个Bean实例，使用该属性指定引用Bean的id
> + value：如果Map value是基本类型或字符串
> + value-ref：如果Map value是容器中的另外一个Bean实例，使用该属性指定引用Bean的id
> 

SpringIoc容器支持集合的合并，子Bean中的集合属性值可以从其父Bean中的集合属性值继承和覆盖而来

``` xml
<beans>
  <bean id="parent" abstract="true" class="org.learn.impl.ComplexObject">
    <property name="adminEmails">
      <props>
        <prop key="admin"/>admin@163.com</prop>
        <prop key="support"/>support@163.com</prop>
      </props>
    </property>
  </bean>
  <bean id="child" parent="parent">
    <property name="adminEmails">
      <props>
        <prop key="sales"/>sales@163.com</prop>
        <prop key="support"/>support@163.com</prop>
      </props>
    </property>
  </bean>
</beans>
```
此时child Bean的adminEmails属性值通过继承和覆盖变为三个

### 组合属性
``` xml
<bean id="chinese" class="org.learn.impl.Chinese">
  <property name="foo.bar.x.y" value="xxxx"/>
</bean>
```
对于这种注入组合属性值的形式，组合属性只有最后一个属性才调用setter方法，前面各属性实际上对应于调用getter方法。故除最后一个属性外，其他属性均不能为null

## Spring提供的Java配置管理
``` java
@Configuration
public class AppConfig
{
  @Value("孙悟空") String personName;
  
  @Bean("name=chinese")
  public Person person()
  {
    Chinese p = new Person();
    p.setName(personName);
    p.setAxe(stoneAxe());
    return p;
  }
  
  @Bean(name="stoneAxe")
  public Axe stoneAxe()
  {
    return new StoneAxe();
  }
  
  @Bean(name="steelAxe")
  public Axe steelAxe()
  {
    return new SteelAxe;
  }
}
```
> + @Configuration：用于修饰一个Java配置类
> + @Bean：用于修饰一个方法，将该方法的返回值定义成容器的一个Bean，可通过那么属性指定该 Bean 的 Id，name 属性可省略Id默认为方法名
> + @Value:用于修饰一个Field，用于为该Field配置一个值，相当于配置一个变量
> 
一旦使用了Java配置类来管理Spring容器中的Bean及其依赖关系，此时就需要如下方式来创建Spring容器：
``` java
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
```

> + @Import：修饰一个Java配置类，用于向当前Java配置类中导入其他Java配置类
> + @Scope：用于修饰一个方法，指定该方法对应的Bean的作用域
> + @Lazy：用于修饰一个方法，指定该方法对应的Bean是否需要延迟初始化
> + @DependsOn：用于修饰一个方法，指定在初始化该方法对应的Bean之前先初始化指定的Bean
> 

可混合使用XML配置文件和Java配置类
> 1. 如果以XML配置为主，就需要让XML配置文件能加载Java类配置。在XML中增加如下代码:  
> 
> 	``` xml
>   <!-- 导入其他 Bean 配置 -->
>   <import resource="beans.xml"/>
>   <context:annotation-config/>
>   <!-- 加载Java配置类-->
>   <bean class="org.learn.app.config.AppConfig"/>
> ```
> 
> 2. 如果以Java配置为主，就需要让Java配置类能加载XML配置。需要使用@ImportResource注解，导入指定的XML配置文件：
> ​
> ``` java 
>  @Configuration
>  @ImportResource("classpath:/beans.xml")
>  public class AppConfig
>  {
>    .....
>  }
> ```

## 创建Bean的3种方式
### 使用构造器创建Bean实例
### 使用静态工厂创建Bean
使用静态工厂方法创建Bean实例时，class属性指向静态工厂类，Spring通过该属性知道是哪个工厂类来创建Bean实例。使用factory-method指定工厂方法，此方法必须是静态的。如果静态工厂需要参数，则使用\<constructor-arg.../\>元素传入。
``` xml
<bean id="dog" class="org.learn.app.BeingFactory" factory-method="getBeing">
    <!-- 配置静态工厂方法的参数 -->
    <constructor-age value="dog"/>
    <!-- 驱动Spring以“我是狗”为参数来执行dog的setMsg方法 -->
    <property name="msg" value="我是狗"/>
</bean>
```
在这个过程中，Spring不再负责创建Bean实例，Bean实例时由用户提供的静态工厂类提供创建的。当静态工厂方法创建了Bean实例后，Spring依然可以管理该Bean实例的的依赖关系，包括为其注入所需的依赖Beean、管理其生命周期等。

### 调用实例工厂方法创建Bean
使用实例工厂方法时，配置Bean实例的\<bean.../\>元素无须class属性，因为Spring容器不再直接实例化该Bean,Spring容器仅仅调用实例工厂的工厂方法，工厂方法负责创建Bean实例。需指定如下参数：
> + factory-bean：该属性的值为工厂Bean的id
> + factory-method：该属性指定实例工厂的工厂方法  
> 

如果需要在调用实例方法时传入参数，则使用\<constructor-arg...\>元素指定参数值

``` xml
<bean id="personFactory" class="org.learn.factory.PersonFactory"/>
<bean id="chinese" factory-method="personFactory" factory-method="getPerson">
    <constructor-arg value="chin"/>
</bean>
</bean>
```

## 深入理解容器中的Bean
### 抽象Bean与子Bean
将多个\<bean.../\>配置中相同的信息提取出来，指定\<bean.../\>中的abstract属性为tru,集中成配置模板---这个模板不是真正的Bean，不能被实例化。抽象Bean的价值在于被继承，抽象Bean通常作为父Bean被继承。抽象Bean可不指定class属性。

``` xml
<beans>
    <bean id="personTemp" abstract="true">
        <property name="name" value="张三"/>
        <property name="axe" ref="steelAxe"/>
    </bean>
    <bean id="chinese" class="org.learn.impl.Chinese" parent="personTemp"/>
    <bean id="american" class="org.learn.impl.Chinese" parent="personTemp"/>
```
当子Bean指定的配置信息与父Bean模板所指定的配置信息不一致时，子Bean所指定的配置信息将覆盖父Bean所指定的配置信息。子Bean无法从父Bean中继承如下属性：depends-on、autowire、singleton、scope、lazy-init。
如果父Bean指定了class属性，那么子Bean连class属性都可以省略，子Bean将采用与父Bean相同的实现类。
> 注:此处的抽象与继承和Java中的抽象继承无关，只是配置文件模板化、复用。

### 容器中的工厂Bean
FactoryBean接口是工厂Bean的标准接口，把工厂Bean（实现BeanFactory接口的Bean）部署在容器中之后，如果程序通过getBean()方法来获取它时，容器返回的不是FactoryBean实现类的实例，而是返回FactoryBean的产品（即该工厂Bean的getObject()方法的返回值）。
FactoryBean接口提供如下三个方法:
``` Java
T getObject(); //实现该方法负责返回该工厂Bean生成的Java实例
Class<?> getObjectType(); //实现该方法返回该工厂Bean生成的Java实例的实现类
boolean isSingleton(); //实现该方法表示该工厂Bean生成的Java实例是否为单例模式 
```
配置FactoryBean与配置普通Bean的定义没有区别，但当程序向Spring容器请求该Bean时，容器返回该BeanFactoryBean的产品，而不是返回该FactoryBean本身。
当程序需要获得FactoryBean本身时，并不是直接请求Bean id，而是在Bean id前增加&符号，容器则返回FactoryBean本身，而不是它生产的Bean。

### 获得Bean自身的id
如果开发Bean时需要预知该Bean的配置id,则可实现BeanNameAware接口，通过该接口即可提前预知该Bean的配置id。BeanNameAware提供如下方法：
``` Java 
setBeanName(String name); //该方法的name参数就是Bean的配置id
```
此接口的setter方法会由Spring容器自动调用，将部署该Bean的id属性作为参数传入。

### 强制初始化Bean
为了显式指定被依赖Bean在目标Bean之前初始化，可以使用depends-on属性，该属性可以在初始化主调Bean之前，强制初始化一个或多个Bean

### 依赖关系注入之后的行为
Spring提供两种方式在Bean全部属性设置成功后执行特定行为：
> + 使用init-method属性
>   使用init-method属性指定某个方法在Bean全部依赖关系设置结束后自动执行。使用这种方式不需要将代码将Spring的接口耦合在一起，代码污染小。***推荐使用***
> + 实现InitializingBean接口
>   达到同样的效果，要求Bean必须继承InitializingBean接口，该接口提供一个方法 ---void afterPropertiesSet() throws Excepion;\>

如果同时使用两种方式，则Spring容器先执行InitializingBean接口中定义的方法，然后执行init-method属性指定的方法。

### Bean销毁之前的行为
Spring同样提供两张方式定制Bean实例销毁之前的特定行为：
> + 使用destory-method属性
>   使用destory-method属性指定某个方法在Bean销毁之前自动执行。使用这种方式不需要将代码将Spring的接口耦合在一起，代码污染小。***推荐使用***
> + 实现DisposableBean接口
>   可以实现同样的效果，但必须实现DisposableBean接口，该接口提供void destory() throw Exception;

如果同时使用两种方式，则Spring容器先执行DisposableBean接口中定义的方法，然后执行destory-method属性指定的方法。

如果处于一个非Web应用的环境下，为了让Spring容器优雅地关闭，并调用singleton Bean相应的析构回调方法，则需要在JVM里注册一个关闭钩子。
``` Java
public class BeanTest
{
    public static void main(String[] args)
    {
        AbstractApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");
        Person p = ctx.getBean("chinese", Person.class);
        p.useAxe();
        
        //为Spring容器注册关闭钩子
        ctx.registerShutdownHook();
    }
}
```
如果在\<beans.../\>中指定了default-init-method="init"，意味着只要Spring容器中的Bean实例具有init()方法，Spring就会在该Spring的所有依赖关系被设置之后，自动调用该Bena实例的init方法。

### 协调作用域不同的步的Bean
当prototype作用域的Bean依赖singleton作用域的Bean时，使用Spring提供的依赖注入进行管理即可。
当singleton作用域的Bean依赖prototype作用域的Bean时，一旦singleton Bean初始化完成，它就持有了一个prototype Bean，容器再也不会为singleton Bean执行注入了。解决此问题有如下方法：
> + 放弃依赖注入：singleton作用域的Bean每次需要prototype作用域的Bean时，主动向容器请新的Bean实例，即可保证每次注入的prototype Bean实例都是最新的实例
> + 利用方法注入

使用lookup方法注入可以让Spring容器重写容器中Bean的抽象或具体方法，返回查找容器中其他Bean的结果，被查找的Bean通常是一个non-singleton Bean（尽管可以是一个singleton的）。Spring通过使用JDK动态代理或cglib库修改客户端的二进制码，从而实现上述要求。
为了实现lookup方法注入，大致需要如下两步：
> 1. 将调用者Bean的实现类定义为抽象类，并定义一个抽象方法来获取被依赖的Bean
> 2. 在\<bean.../\>元素中添加\<lookup-method.../\>子元素让Spring为调用者Bean的实现类实现指定的抽象方法。

\<lookup-method.../\>元素需要如下两个属性：
> + name：指定需要让Spring实现的方法
> + bean：指定Spring实现该方法的返回值

``` Java
public abstract class Chinese 
{
    private Dog dog;
    public abstract Dog getDog();
    public void hunt()
    {
        getDog().run();
    }
}
```

``` xml
<bean id="personTemp" abstract="true">
    <!-- Spring只要检测到lookup-method元素，Spring会自动为该元素的name属性指定的方法提供实现体 -->
    <lookup-method name="getDog" bean="gunDog"/>
</bean>
<bean id="gunDog" class="org.leanrn.impl.GunDog" scope="prototype">
    <property name="name" value="旺财"/>
</bean>
```
Spring会采用运行时动态增强的方式来实现\<lookup-method.../\>元素所指定的抽象方法，如果目标抽象类实现过接口，Spring会采用JDK动态代理来实现该抽象类，并为之实现抽象方法；如果目标抽象类没有实现过接口，Spring会采用cglib实现该抽象类，并为之实现抽象方法。

要保证\<lookup-method.../\>方法注入每次都产生新的Bean实例，必须将目标Bean部署成prototype作用于；否则，如果容器中只有一个被依赖的Bean实例，即使采用lookup方法注入，每次也依然返回同一个Bean实例。

## 高级依赖关系配置
### 获取其他Bean的属性值
PropertyPathFactoryBean用来获取目标Bean的属性值（实际上就是它的getter方法的返回值），获得的值可注入其他Bean，也可直接定义成新的Bean。使用PropertyPathFactoryBean调用其他Bean的getter方法需要指定如下信息：
> 调用哪个对象。由PropertyPathFactoryBean的setTargetObject(Object targetObject)方法指定；
> 调用哪个getter方法。由PropertyPathFactoryBean的setPropertyPath(String propertyPath)方法指定；

``` xml
<!-- target bean to be referenced by name -->
 <bean id="tb" class="org.springframework.beans.TestBean" singleton="false">
   <property name="age" value="10"/>
   <property name="spouse">
     <bean class="org.springframework.beans.TestBean">
       <property name="age" value="11"/>
     </bean>
   </property>
 </bean>

 <!-- will result in 12, which is the value of property 'age' of the inner bean -->
 <bean id="propertyPath1" class="org.springframework.beans.factory.config.PropertyPathFactoryBean">
   <property name="targetObject">
     <bean class="org.springframework.beans.TestBean">
       <property name="age" value="12"/>
     </bean>
   </property>
   <property name="propertyPath" value="age"/>
 </bean>

 <!-- will result in 11, which is the value of property 'spouse.age' of bean 'tb' -->
 <bean id="propertyPath2" class="org.springframework.beans.factory.config.PropertyPathFactoryBean">
   <property name="targetBeanName" value="tb"/>
   <property name="propertyPath" value="spouse.age"/>
 </bean>

 <!-- will result in 10, which is the value of property 'age' of bean 'tb' -->
 <bean id="tb.age" class="org.springframework.beans.factory.config.PropertyPathFactoryBean"/>
```
可以使用\<util:property-path...\>作为PropertyPathFactoryBean的简化配置，需要如下两个属性:
> + id：该属性指定将getter方法的返回值定义成名为id的Bean的实例
> + path：该属性指定将哪个Bean实例、哪个属性（支持复合属性）暴露出来

``` xml
 <!-- will result in 10, which is the value of property 'age' of bean 'tb' -->
 <util:property-path id="name" path="testBean.age"/>
```

PropertyPathFactoryBean就是工厂Bean，在这种配置方式下，配置PropertyPathFactoryBean工厂Bean时指定的id属性，并不是该Bean的唯一标识，而是用于指定属性表达式的值。

### 获取Field值
通过FieldRetrievingFactoryBean类，可访问类的静态Field或对象的实例Field值。FieldRetrievingFactoryBean获得指定Field的值之后，即可将获取的值注入其他Bean，也可直接定义成新的Bean。使用FieldRetrievingFactoryBean分一下两种情况：
> + 如果要访问的Field是静态Field,则需要指定:
> > 调用哪个类。由FieldRetrievingFactoryBean的setTargetClass(String targeClass)方法指定；
> > 访问哪个Field。由FieldRetrievingFactoryBean的setTargetField(String targetField)方法指定。
> + 如果要访问的Field是实例Field（要求实例Field以public修饰），则需要指定：
> > 调用哪个对象。由FieldRetrievingFactoryBean的setTargetObject(Object targetObject)方法指定
> > 访问哪个Field。由FieldRetrievingFactoryBean的setTargetField(String targetField)方法指定。
>  FieldRetrievingFactoryBean还提供了一个setStaticField(String staticField)方法，该方法可同时指定获取哪个类的哪个静态Field的值。

``` xml
 // standard definition for exposing a static field, specifying the "staticField" property
 <bean id="myField" class="org.springframework.beans.factory.config.FieldRetrievingFactoryBean">
   <property name="staticField" value="java.sql.Connection.TRANSACTION_SERIALIZABLE"/>
 </bean>

 // convenience version that specifies a static field pattern as bean name
 <bean id="java.sql.Connection.TRANSACTION_SERIALIZABLE"
       class="org.springframework.beans.factory.config.FieldRetrievingFactoryBean"/>
```

`<util:constant.../>`元素可作为FieldRetrievingFactoryBean访问静态Field的简化配置，使用该元素时可指定如下两个属性：
> + id：该属性指定将静态Field的值定义成名为id的Bean实例
> + static-field：该属性指定访问哪个类的哪个静态Field

``` xml
<util:constant static-field="java.sql.Connection.TRANSACTION_SERIALIZABLE"/>
```

### 获取方法返回值
通过MethodInvokingFactoryBean工厂Bean可以调用任意类的类方法，也可调用任意对象的实例方法，如果调用的方法有返回值，既可将该指定方法的返回值定义成容器中的Bean，也可将指定方法的返回值注入给其他Bean。使用MethodInvokingFactoryBean分一下两种情况：
> + 如果希望调用的方法时静态方法，则需要指定：
> > 调用哪个类。通过MethodInvokingFactoryBean的setTargetClass(String targetClass)方法指定；
> > 调用哪个方法。通过MethodInvokingFactoryBean的setTargetMethod(String targetMethod)方法指定；
> > 调用方法的参数。通过MethodInvokingFactoryBean的 setArguments(Object[] arguments)方法指定。
> + 如果希望调用的方法是实例方法，则需要指定：
> > 调用哪个对象。通过MethodInvokingFactoryBean的 setTargetObject(Object targetObject)指定；
> > 调用哪个方法。通过MethodInvokingFactoryBean的setTargetMethod(String targetMethod)方法指定；
> > 调用方法的参数。通过MethodInvokingFactoryBean的 setArguments(Object[] arguments)方法指定。

bean定义的一个示例（在基于XML的bean工厂定义中），它使用此类来调用静态工厂方法：
``` xml
 <bean id="myObject" class="org.springframework.beans.factory.config.MethodInvokingFactoryBean">
   <property name="staticMethod" value="com.whatever.MyClassFactory.getInstance"/>
 </bean>
```
调用静态方法然后使用实例方法获取Java系统属性的示例
``` xml
 <bean id="sysProps" class="org.springframework.beans.factory.config.MethodInvokingFactoryBean">
   <property name="targetClass" value="java.lang.System"/>
   <property name="targetMethod" value="getProperties"/>
 </bean>

 <bean id="javaVersion" class="org.springframework.beans.factory.config.MethodInvokingFactoryBean">
   <property name="targetObject" ref="sysProps"/>
   <property name="targetMethod" value="getProperty"/>
   <property name="arguments" value="java.version"/>
 </bean>
```

## 基于XML Schema 的简化配置方式
### 使用 p:命名空间简化配置(简化设值注入)
需要导入 XML Schema 里的 p:命名空间`xmlns:p="http://www.springframework.org/shema/p"`
``` xml
<bean id="chinese" class="org.learn.impl.Chinese" p:age="29" p:axe-ref="stoneAxe"/>
<bean id="stoneAxe" class="org.learn.impl.StoneAxe"/>
```
### 使用c:命名控件简化配置（简化构造注入）
需要导入 XML Schema 里的 c:命名空间`xmlns:c="http://www.springframework.org/shema/c`
``` xml
<bean id="chinese" class="org.learn.impl.Chinese" c:age="29" c:axe-ref="stoneAxe"/>
<bean id="stoneAxe" class="org.learn.impl.StoneAxe"/>
```
可使用 c\_N 的方式指定构造器参数位置。
```
<bean id="chinese" class="org.learn.impl.Chinese" c:_0="29" c:_1-ref="stoneAxe"/>
```

### 使用 util:命名空间简化配置
需要导入 XML Schema 里的 util:命名空间 `xmlns:util="http://www.springframework.org/shema/util`
> constant：该元素用于获取指定类的静态 Field
> property-path：该元素用于获取指定对象的 getter 方法的返回值
> list：该元素用于定一个一个 List Bean，支持使用\<value.../\>、\<ref.../\>、\<bean.../\>等子元素来定义 List 集合元素。该标签支持如下三个属性：
> > + id :该属性指定定一个名为 id 的 List Bean 实例
> > + list-class：该属性指定 Spring 通过那个 List 实现类来创建 Bean 实例。默认使用 ArrayList
> > + scope：指定该 List Bean 的作用域
> > 
> 
> set：该元素用于定一个一个 Set Bean，支持使用\<value.../\>、\<ref.../\>、\<bean.../\>等子元素来定义 Set 集合元素。该标签支持如下三个属性：
> > + id :该属性指定定一个名为 id 的 Set Bean 实例
> > + set-class：该属性指定 Spring 通过那个 Set 实现类来创建 Bean 实例。默认使用 HashSet
> > + scope：指定该 Set Bean 的作用域
> 
> map：该元素用于定一个一个 Map Bean，支持使用\<entry.../\>元素来定义 Map 的键值对。该标签支持如下三个属性：
>  \> + id :该属性指定定一个名为 id 的 Map Bean 实例
> > + map-class：该属性指定 Spring 通过那个 Map 实现类来创建 Bean 实例。默认使用 HashMap
> > + scope：指定该 Map Bean 的作用域
> 
> properties：该元素用于加载一份资源文件，并根据加载的资源文件创建一个 Properties Bean 实例。该标签支持如下三个属性：
> > + id :该属性指定定一个名为 id 的 Properties Bean 实例
> > + location：指定资源文件的位置
> > + scope：指定该 Properties Bean 的作用域

### SpEL 语法详述
1. 直接量表达式  
	`{3.14159}` `{'Hello'}` `{false}`
2. 引用 bean、属性和方法
 SpEL 通过 ID 引用其它的 Bean`{sgtPeppers}`<br>
 引用 Bean 中的属性`{sgtPeppers.artist}`<br>
 引用 Bean 中的方法`{sgtPeppers.selectArtist()}`
3. 安全导航  
	`{foo?.bar}` 先判断 foo是否为 null，如果 foo 为 null 就直接返回 null
4. 在表达式中使用类型 
	`{T(java.lang.math).PI}`这里的 T()运算符的结果会是一个 Class 对象，表示访问类作用域的方法和常量
5. 运算符    
	`{2 * T(java.lang.Math).PI * circle.radius}`
	`{disc.title ?: 'Hello World'}`此处的?:表示判断 disc.title 是否为 null，如果为 null 的话指定默认值为 Hello World
6. 计算集合
	`{jukebox.songs[4].title}` 表示查询 jukebox 中 songs 集合的第5个元素的 title,此处[]中表示第几个元素
	`{jukebox.songs.?[artist eq 'Hello']}` 此处的.?[]运算符表示对指定集合过滤，得到集合的一个子集
	`{jukebox.songs.^[artist eq 'Hello']}`和`jukebox.songs.?[artist eq 'Hello']`此处中.^[]和.$[]分别表示查询集合中的第一个匹配项和最后一个匹配项
	`jukebox.songs.![title]`此处的.![]是投影运算符，表示将集合的每个成员中选择特定的属性放到另外一个集合中。





