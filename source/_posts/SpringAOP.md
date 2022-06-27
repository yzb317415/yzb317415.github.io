---
title: SpringAOP
categories: 
	- [JAVA后端开发, Spring]
tags: 
	- Spring
	- 学习笔记
date: 2018-12-25 00:00:00
---

# Spring的AOP
AOP实现可分为两类
> + 静态AOP实现
>   AOP框架在编译阶段对程序进行修改，即实现对目标类型的增强，生成静态的AOP代理类（生成的*.class文件已经被改掉了，需要使用特定的编译器）。以AspectJ代表。
> + 动态AOP实现
>   AOP框架在巡行阶段动态生成AOP代理（在内存中以JDK动态代理或cglib动态地生成AOP代理类），以实现对目标对象的增强。以SpringAOP为代表

一般来说，静态AOP实现具有较好的性能，但需要使用特殊的编译器。动态AOP实现是纯Java实现，因此无须特殊的编译器，但是通常性能较差。

## AOP的基本概念
术语:
> - 切面（Aspect）:切面用于组织多个Advice，Advice放在切面中定义
> - 连接点（Joinpoint）：程序执行过程中明确的店，如方法的调用，或者异常的抛出。在SpringAOP中，连接点总是方法的调用
> - 增强处理（Advice）：AOP框架在特定的切入点执行的增强处理。处理有“around”、“before”、“after”等类型
> - 切入点（Ponitcut）：可以插入增强处理的连接点。简而言之，当某个连接点满足指定要求时，该连接点将被添加增强处理。该连接点也就变成了切入点
> - 引入：将方法或字段添加到被处理的类中
> - 目标对象：被AOP框架进行增强处理的对象。如果AOP框架采用的是动态代理AOP实现，那么该对象就是一个被代理的对象
> - AOP代理：AOP框架创建的对象，简单地说，代理就是对目标对象的加强
> - 织入：将增强处添加到目标对象中，并创建一个被增强的对象的过程就是织入

Spring中默认使用Java动态代理来创建AOP代理，也可使用cglib代理（需要代理类而不是代理接口的时候）。
SpringAOP中只支持使用方法调用作为连接点。

AOP编程的步骤：
> - 定义普通业务组件
> - 定义切入点，一个切入点可能横切多个业务组件
> - 定义增强处理，增强处理就是在 AOP 框架为普通业务组件织入的处理动作

## 基于注解的“零配置”方式
配置启动AspectJ自动代理的三种方式
> 1. XML方式:在`<beans.../>`中增加`<aop:aspectj-autoprox/>`子元素
> 2. 使用注解方式：使用@EnableAspectAutoProxy修饰使用@Configuration修饰的Java配置类
> 3. 通过Bean后处理器：配置AnnotationAwareAspectJAutoProxyCreator的Bean后处理器，该Bean后处理器将会为容器中符合条件的Bean生成AOP代理

### 定义切面Bean
当启用了@AspectJ支持后，只要在Spring容器中配置一个带@Aspect注解的Bean，Spring将会自动识别该Bean，并将该Bean作为切面处理
在Spring容器中配置切面Bean，与配置普通Bean没有任何区别
``` java
@Aspect
public class LogAspect
{
   // 定义该类的其他内容 
}
```
当使用@Aspect修饰一个Java类之后，Spring将不会把该Bean当成组件Bean处理，因此负责自动增强的Bean将会略过该Bean，不会对该Bean进行任何增强处理

### 定义Before增强处理
在一个切面类里使用@Before修饰一个方法时，该方法将作为Befo增强处理。使用@Befo修饰时，通常需要指定一个value属性，该属性指定一个切入点表达式（既可以是一个已有的切入点，也可以直接定义切入点表达式），用于指定该增强处理将被织入哪些切入点
``` java
@Aspect
public class LogAspect
{
    @Before("execution(* org.learn.service.impl.*.*(..)")
    public void authority()
    {
        System.out.println("模拟执行权限检查");
    }
}
```
使用Befor增强处理只能在目标方法执行之前织入增强，如果Before增强处理没有特殊处理，目标方法总会自动执行，如果Before处理需要组织目标方法的执行，可通过抛出一个异常实现。Before增强处理处理执行时，目标方法还没获得执行的机会，所以Before增强处理无法访问目标方法的返回值

### 定义AfterReturning增强处理
使用@AfterReturning可修饰AfterReturning增强处理，AfterReturning增强处理将在目标正常完成后被织入
使用@AfterReturning注解可指定如下两个常用属性：
> - pointcut/value：这两个属性的作用是一样的，它们都是用于指定该切入点对应的切入表达式。当指定了pointcut属性之后，value属性值将会被覆盖
> - returning：该属性值指定一个形参名，用于表示Advice方法中可定义与此同名的形参，该形参可用于访问目标方法的返回值。除此之外，在Advice方法中定义该形参（代表目标方法的返回值）是指定的类型，会限制目标方法必须返回指定类型的值
> 

``` Java
@Aspect
public class LogAspect
{
	// 匹配org.crazyit.app.service.impl包下所有类的、
	// 所有方法的执行作为切入点
	@AfterReturning(returning="rvt"
		, pointcut="execution(* org.crazyit.app.service.impl.*.*(..))")
	// 声明rvt时指定的类型会限制目标方法必须返回指定类型的值或没有返回值
	// 此处将rvt的类型声明为Object，意味着对目标方法的返回值不加限制
	public void log(Object rvt)
	{
		System.out.println("获取目标方法返回值:" + rvt);
		System.out.println("模拟记录日志功能...");
	}
}
```

### 定义AfterThrowing增强处理
使用@AfterThrowing可修饰AfterThrowing增强处理，AfterThrowing增强处理将主要用于处理程序中未处理的异常。
使用@AfterThrowing注解可指定如下两个常用属性：
> - pointcut/value：这两个属性的作用是一样的，它们都是用于指定该切入点对应的切入表达式。当指定了pointcut属性之后，value属性值将会被覆盖
> - thorwing：该属性值指定一个形参名，用于表示Advice方法中可定义与此同名的形参，该形参可用于访问目标方法抛出的异常。除此之外，在Advice方法中定义该形参（代表目标方法抛出的异常）时指定的类型，会限制目标方法必须抛出指定类型的异常

``` java
// 定义一个切面
@Aspect
public class RepairAspect
{
	// 匹配org.crazyit.app.service.impl包下所有类的、
	// 所有方法的执行作为切入点
	@AfterThrowing(throwing="ex"
		, pointcut="execution(* org.crazyit.app.service.impl.*.*(..))")
	// 声明ex时指定的类型会限制目标方法必须抛出指定类型的异常
	// 此处将ex的类型声明为Throwable，意味着对目标方法抛出的异常不加限制
	public void doRecoveryActions(Throwable ex)
	{
		System.out.println("目标方法中抛出的异常:" + ex);
		System.out.println("模拟Advice对异常的修复...");
	}
}
```

catch捕捉意味着完全处理该异常，如果catch块中没有重新抛出新异常，则该方法可以正常结束；而AfterThrowing处理虽然处理了该异常，但它不能完全处理该异常，该异常依然会传播到上一级调用者

### After增强处理
Spring中提供的After增强处理，它与AfterReturning增强处理类似，但也有区别
> + AfterReturning增强处理只有在目标方法成功完成后才会被织入
> + After增强处理不管目标方法如何借宿（包括成功完成和遇到异常终止两种情况），它都会被织入

After增强处理有点类似于finally块
使用@After注解时需要指定value属性，该属性用于指定该增强处理被织入的切入点，既可以是一个已有的切入点，也可直接指定切入点表达式
``` java
// 定义一个切面
@Aspect
public class ReleaseAspect
{
	// 匹配org.crazyit.app.service包下所有类的
	// 所有方法的执行作为切入点
	@After("execution(* org.crazyit.app.service.*.*(..))")
	public void release()
	{
		System.out.println("模拟方法结束后的释放资源...");
	}
}
```

### Around增强处理
@Around注解用于修饰Around增强处理，Around增强处理是功能强大的增强处理，它近似等于Before增强处理和AfterReturning增强处理的总和，Around增强处理既可在执行目标方法之前织入增强动作，也可在执行目标方法之后织入增强动作

Around增强处理可以决定目标方法在什么时候执行，如何执行，甚至可以完全阻止目标方法的执行。Around增强处理可以改变执行目标方法的参数值，也可以改变执行目标方法之后的返回值。

Around通常需要在线程安全的环境下使用。因此如果普通的Before增强处理、AfterReturning增强处理就能解决的问题，则没有必要使用Around增强处理了。如果需要目标方法执行之前和之后共享某种状态数据，则应该考虑使用Around增强处理；尤其是需要改变目标方法的返回值时，则中能使用Around增强处理

Around增强处理使用@Around标注，需要指定一个value属性指定该增强处理被织入的切入点

当定义一个Around增强处理方法时，该方法的第一个形参必须是**ProceedingJoinPoin**类型（至少包含一个形参），在增强方法体内，调用ProceedingJoinPoin参数的proceed()方法才会执行目标方法，如果程序没有调用ProceedingJoinPoin的proceed()方法，则目标方法不会被执行。
当调用ProceedingJoinPoin参数的proceed方法时，还可以传入一个Object[]对象最为参数，该数组中的值将被传入目标方法作为执行方法的实参

``` java
// 定义一个切面
@Aspect
public class TxAspect
{
	// 匹配org.crazyit.app.service.impl包下所有类的、
	// 所有方法的执行作为切入点
	@Around("execution(* org.crazyit.app.service.impl.*.*(..))")
	public Object processTx(ProceedingJoinPoint jp)
		throws java.lang.Throwable
	{
		System.out.println("执行目标方法之前，模拟开始事务...");
		// 获取目标方法原始的调用参数
		Object[] args = jp.getArgs();
		if(args != null && args.length > 1)
		{
			// 修改目标方法的第一个参数
			args[0] = "【增加的前缀】" + args[0];
		}
		// 以改变后的参数去执行目标方法，并保存目标方法执行后的返回值
		Object rvt = jp.proceed(args);
		System.out.println("执行目标方法之后，模拟结束事务...");
		// 如果rvt的类型是Integer，将rvt改为它的平方
		if(rvt != null && rvt instanceof Integer)
			rvt = (Integer)rvt * (Integer)rvt;
		return rvt;
	}
}
```

### 访问目标方法的参数
访问目标方法最简单的做法就是定义增强处理方法时将第一个参数定义为JoinPoint类型，当该增强处理方法被调用时，该JoinPoint参数就代表了织入增强处理的连接点。JoinPoint里包含如下几个常用方法：
> `Object[] getArg()`：返回执行目标方法时的参数
> `Signature getSignature()`：返回被增强的方法的相关信息
> `Object getTarget()`：返回被织入增强处理的目标参数
> `Object getThis()`：返回AOP框架为目标对象生成的代理对象

ProceedingJoinPoin就是JoinPoint方法的子类

``` java
// 定义一个切面
@Aspect
public class FourAdviceTest
{
	// 定义Around增强处理
	@Around("execution(* org.crazyit.app.service.impl.*.*(..))")
	public Object processTx(ProceedingJoinPoint jp)
		throws java.lang.Throwable
	{
		System.out.println("Around增强：执行目标方法之前，模拟开始事务...");
		// 访问执行目标方法的参数
		Object[] args = jp.getArgs();
		// 当执行目标方法的参数存在，
		// 且第一个参数是字符串参数
		if (args != null && args.length > 0
			&& args[0].getClass() == String.class)
		{
			// 修改目标方法调用参数的第一个参数
			args[0] = "【增加的前缀】" + args[0];
		}
		//执行目标方法，并保存目标方法执行后的返回值
		Object rvt = jp.proceed(args);
		System.out.println("Around增强：执行目标方法之后，模拟结束事务...");
		// 如果rvt的类型是Integer，将rvt改为它的平方
		if(rvt != null && rvt instanceof Integer)
			rvt = (Integer)rvt * (Integer)rvt;
		return rvt;
	}
	// 定义Before增强处理
	@Before("execution(* org.crazyit.app.service.impl.*.*(..))")
	public void authority(JoinPoint jp)
	{
		System.out.println("Before增强：模拟执行权限检查");
		// 返回被织入增强处理的目标方法
		System.out.println("Before增强：被织入增强处理的目标方法为："
			+ jp.getSignature().getName());
		// 访问执行目标方法的参数
		System.out.println("Before增强：目标方法的参数为："
			+ Arrays.toString(jp.getArgs()));
		// 访问被增强处理的目标对象
		System.out.println("Before增强：被织入增强处理的目标对象为："
			+ jp.getTarget());
	}
	//定义AfterReturning增强处理
	@AfterReturning(pointcut="execution(* org.crazyit.app.service.impl.*.*(..))"
		, returning="rvt")
	public void log(JoinPoint jp , Object rvt)
	{
		System.out.println("AfterReturning增强：获取目标方法返回值:"
			+ rvt);
		System.out.println("AfterReturning增强：模拟记录日志功能...");
		// 返回被织入增强处理的目标方法
		System.out.println("AfterReturning增强：被织入增强处理的目标方法为："
			+ jp.getSignature().getName());
		// 访问执行目标方法的参数
		System.out.println("AfterReturning增强：目标方法的参数为："
			+ Arrays.toString(jp.getArgs()));
		// 访问被增强处理的目标对象
		System.out.println("AfterReturning增强：被织入增强处理的目标对象为："
			+ jp.getTarget());
	}

	// 定义After增强处理
	@After("execution(* org.crazyit.app.service.impl.*.*(..))")
	public void release(JoinPoint jp)
	{
		System.out.println("After增强：模拟方法结束后的释放资源...");
		// 返回被织入增强处理的目标方法
		System.out.println("After增强：被织入增强处理的目标方法为："
			+ jp.getSignature().getName());
		// 访问执行目标方法的参数
		System.out.println("After增强：目标方法的参数为："
			+ Arrays.toString(jp.getArgs()));
		// 访问被增强处理的目标对象
		System.out.println("After增强：被织入增强处理的目标对象为："
			+ jp.getTarget());
	}
}

```

在“进入”连接点时，具有最高优先级的增强处理将被优先织入（所以在给定的两个Befoe增强处理中，优先级高的哪个会先执行）。在“退出”连接点时，具有最高优先级的增强处理会最后被织入（所以在给定的两个After增强处理中，优先级高的那个会后执行）

当不同切面里的两个增强处理需要在用一个连接点被织入时，Spring AOP将以随机的顺序来织入这两个增强处理。如果应用需要制定不同切面类里增强处理的优先级，Spring提供了如下两种解决方案：
> + 让切面类实现org.springframework.core.Ordered接口，实现该接口只需实现一个int getOrder()方法，该方法的返回值越小，则优先级越高
> + 直接使用@Order注解来修饰一个切面类，使用@Order注解时可指定一个int型value属性，该属性值越小，则优先级越高

如果只需要访问目标方法的参数，Spring还提供了一种更简单的方法：可以在程序中使用args切入点表达式来绑定目标方法的参数。如果在一个args表达式中指定了一个或多个参数，则该切入点将只匹配具有相应形参的方法，且目标方法的参数值将百日传入增强处理方法
``` java
@Aspect
public class AccessArgAspect
{
	// 下面的args(arg0,arg1)会限制目标方法必须有2个形参
	@AfterReturning(returning="rvt" , pointcut=
		"execution(* org.crazyit.app.service.impl.*.*(..)) && args(arg0,arg1)")
	// 此处指定arg0、arg1为String类型
	// 则args(arg0,arg1)还要求目标方法的两个形参都是String类型
	public void access(Object rvt, String arg0 , String arg1)
	{
		System.out.println("调用目标方法第1个参数为:" + arg0);
		System.out.println("调用目标方法第2个参数为:" + arg1);
		System.out.println("获取目标方法返回值:" + rvt);
		System.out.println("模拟记录日志功能...");
	}
}
```

使用args表达式有如下两个作用：
> - 提供了一种简单的方式来访问目标方法的参数
> - 对表达式增加额外的限制

### 定义切入点
SpringAOP只支持将Spring Bean的方法执行作为连接点，所以可以把切入点看成所有能和切入点表达式匹配的Bean的方法
切入点定义包含两部分：
> - 一个切入点表达式：用于指定改切入点和哪些方法进行匹配
> - 一个包含名字和任意参数的方法签名：作为该切入点的名称

在@AspectJ风格的AOP中，切入点签名必须采用一个普通的方法定义（方法体通常为空）来提供，且该方法的返回值必须是void；切入点表达式需要使用@Pointcut注解来标注
``` java
// 使用@Ponitcut注解定义切入点
@Pointcut("execution(* transfer(..))")
// 使用一个返回值为void、方法体为空的方法来命名切入点
public void anyOldTransfer(){}
```

如果需要使用本切面类的切入点，则可使用@Before、@After、@Around等注解定义Advice时，使用pointcut或value属性值引用已有的切入点
``` java
@AfterReturning(pointcut="mypoint()",returning="relval")
public void writelog(String msg, Object retval)
{
    ...
}
```

如果需要使用其他切面类中的切入点，则其他切面类中的切入点不能使用private修饰。而且在使用@Before、@After、@Around等注解中的pointcut或value属性值引用已有的切入点时，必须添加类名前缀

### 切入点指示符
SpringAOP一共支持如下几种方法的连接点：
> - execution：用于匹配执行方法的连接点，execution表达式的格式如下：
>   
>   `execution(modifiers-pattern? ret-type-pattern declaring-type-pattern? name-pattern(param-pattern) throw-pattern?)`
>   > - modifiers-pattern：指定方法的修饰符，支持通配符，可省略
>   > - ret-type-pattern：指定方法的返回值类型，支持通配符，可以使用“\*”通配符来匹配所用的返回值类型
>   > - declaring-type-pattern：指定方法所属的类，支持通配符，可省略
>   > - name-pattern：指定匹配指定的方法名，支持通配符，可以使用“\*”通配符来匹配所有方法
>   > - param-pattern：指定方法声明中的形参列表，支持两个通配符，“\*”代表一个任意类型的参数，“..”代表零个或多个任意类型的参数
>   > - throw-pattern：指定方法声明抛出的异常，支持通配符，可省略
> 
>   ``` java
>   // 匹配任意public方法的执行
>   execution(public * * (..))
>   // 匹配任何方法名以“set”开始的方法的执行
>   execution(* set*(..))
>   // 匹配AccountServiceImpl中任意方法的执行
>   execution(* org.learn.impl.AccountServiceImpl.* (..))
>   // 匹配org.learn.impl.包中任意类的任意方法
>   execution(* org.learn.impl.*.* (..))
>   ```
> - within：用于限定匹配特定类型的连接点，在SpringAOP中只是方法执行的连接点
>
>   ``` java
>   // 在org.learn.app.service包中的任意连接点（在SpringAOP中只是方法执行的连接点）
>   whitin(org.learn.app.service.*) 
>   // 在org.learn.app.service包或其子包中的任意连接点（在SpringAOP中只是方法执行的连接点）
>   whitin(org.learn.app.service..*) 
>   ```
> - this：用于限定AOP代理必须是指定类型的实例，匹配该对象的所有连接点。在SpringAOP中只是方法执行的连接点
> ``` java
> // 匹配实现了 org.learn.app.service.AccountService接口的 AOP 代理的所有连接点
> this(org.learn.app.service.AccountService)
> ```
> 
> - target 用于限定目标对象必须是指定类型的实例，匹配该对象的所有连接点。
> ``` java
> // 匹配实现了 org.learn.app.service.AccountService接口的目标对象的所有连接点
> this(org.learn.app.service.AccountService)
> ```
> 
> - args：用于对连接点的参数类型进行限制，要求参数类型必须是知道那个类型
> - bean：用于限定值匹配指定 Bean 实例内的连接点
> ``` java
> // 匹配 tradeService Bean 实例内方法执行的连接点
> bean(tradeService)
> // 匹配名字以 Serivice 结尾的 Bean 实例内方法执行的连接点
> bean(*Service)
> ```

### 组合切入点表达式
Spring 支持如下三个逻辑运算符来组合切入点表达式
> - `&&`：要求连接点同时匹配两个切入点表达式
> - `||`：要求连接点匹配任意个切入点表达式
> - `!:`：要求连接点不匹配指定的切入点表达式

## 基于 XML 配置文件的管理方式
在 Spring 配置文件中，所有的切面、切入点和增强处理都必须定义在`<aop:config.../>`元素内部。`<beans.../>`元素下可以包含多个`<aop:config.../>`元素，一个`<aop:config>`可以包含 pointcut、advisor 和 aspect 元素，且这三个元素必须按照此顺序来定义。

### 配置切面
因为切面 Bean 可以当成一个普通的 Spring Bean 来配置，所以完全可以为该切面 Bean 配置依赖注入。当切面 Bean 定义完成后，通过在`<aop:aspect>`元素中使用 ref 属性来引用该 Bean，就可将该 Bean 转换成一个切面 Bean了。

配置`<aop:aspect>`元素时可以指定如下三个属性：
> + id：定义该切面的标识名
> + ref：用于将 ref 属性所引用的普通 Bean 转换为切面 Bean
> + order：指定切面 Bean 的优先级，值越小，优先级越高
```
<aop:config>
    <!-- 将容器中的 afterAdiveBean 转换成切面 Bean 
        切面 Bean 的新名称为：afterAdviceAspect -->
    <aop:aspect id="afterAdviceAspect" ref="afterAdviceBean" >
    </aop:aspect>
</aop:config>
<!-- 定义一个普通 Bean 实例，该 Bean 实例将作为 Aspect Bean -->
<bean id="afterAdviceBean" class="lee.AfterAdviceTest"/>
```

### 配置增强处理
使用 XML 配置增强处理依赖如下几个元素：
> + `<aop:before.../>`：配置 Before 增强处理
> + `<aop:after.../>`：配置 After 增强处理
> + `<aop:after-returning.../>`：配置 AfterReturning 增强处理
> + `<aop:after-throwing.../>`：配置 AfterThrowing 增强处理
> + `<aop:around.../>`：配置 Around 增强处理

上面的这些元素都不支持使用子元素，但通常可指定如下属性：
> + pointcut|pointcut-ref：pointcut 属性将指定一个切入表达式，pointcut-ref 属性指定以后的切入点名称，Spring 将在匹配表达式的连接点时织入该增强处理。通常这两个属性只使用一个
> + method：该属性指定一个方法名，指定将切面Bean的该方法转换为增强处理
> + throwing：该属性只对`<aop:after-throwing.../>`元素有效，用于指定一个形参名，AfterThrowing增强处理方法可以通过该形参访问目标方法所抛出的异常
> + returning：该属性只对`<aop:after-returning.../>`元素有效，用于指定一个形参名，AfterReturning增强处理方法可通过该属性访问目标方法的返回值

XML配置方式和@AspectJ注解方式一样支持组合切入点表达式，但XML配置方式不再使用简单的&&、||和!作为组合表达式，而是使用如下三个组合运算符：and、or、not

``` xml
<beans>
    <aop:config>
		<!-- 将fourAdviceBean转换成切面Bean
			切面Bean的新名称为：fourAdviceAspect
			指定该切面的优先级为2 -->
		<aop:aspect id="fourAdviceAspect" ref="fourAdviceBean"
			order="2">
			<!-- 定义一个After增强处理
				直接指定切入点表达式
				以切面Bean中的release()方法作为增强处理方法 -->
			<aop:after pointcut="execution(* org.crazyit.app.service.impl.*.*(..))" 
				method="release"/>
			<!-- 定义一个Before增强处理
				直接指定切入点表达式
				以切面Bean中的authority()方法作为增强处理方法 -->
			<aop:before pointcut="execution(* org.crazyit.app.service.impl.*.*(..))" 
				method="authority"/>
			<!-- 定义一个AfterReturning增强处理
				直接指定切入点表达式
				以切面Bean中的log()方法作为增强处理方法 -->
			<aop:after-returning pointcut="execution(* org.crazyit.app.service.impl.*.*(..))"
				method="log" returning="rvt"/>
			<!-- 定义一个Around增强处理
				直接指定切入点表达式
				以切面Bean中的processTx()方法作为增强处理方法 -->
			<aop:around pointcut="execution(* org.crazyit.app.service.impl.*.*(..))" 
				method="processTx"/>
		</aop:aspect>
		<!-- 将secondAdviceBean转换成切面Bean
			切面Bean的新名称为：secondAdviceAspect
			指定该切面的优先级为1，该切面里的增强处理将被优先织入 -->
		<aop:aspect id="secondAdviceAspect" ref="secondAdviceBean"
			order="1">
			<!-- 定义一个Before增强处理
				直接指定切入点表达式
				以切面Bean中的authority()方法作为增强处理方法 
				且该参数必须为String类型（由authority方法声明中aa参数的类型决定） -->
			<aop:before pointcut=
			"execution(* org.crazyit.app.service.impl.*.*(..)) and args(aa,..)" 
				method="authority"/>
		</aop:aspect>
	</aop:config>
	<!-- 定义一个普通Bean实例，该Bean实例将被作为Aspect Bean -->
	<bean id="fourAdviceBean"
		class="org.crazyit.app.aspect.FourAdviceTest"/>
	<!-- 再定义一个普通Bean实例，该Bean实例将被作为Aspect Bean -->
	<bean id="secondAdviceBean"
		class="org.crazyit.app.aspect.SecondAdviceTest"/>
	<bean id="hello" class="org.crazyit.app.service.impl.HelloImpl"/>
	<bean id="world" class="org.crazyit.app.service.impl.WorldImpl"/>
</beans>

```

### 配置切入点
Spring提供了`<aop:pointcut.../>`元素来定义切入点。当把`<aop:pointcut>`元素作为`<aop:config.../>`的子元素定义时，表明该切入点可被多个切面共享；当把`<aop:pointcut.../>`元素作为`<aop:aspect.../>`子元素定义时，表明该切入点只能在该切面中有效

配置`<aop:pointcut.../>`元素时通常需要指定如下两个属性：
> + id：指定该切入点的标识名
> + expression：指定该切入点关联的切入点表达式

``` xml
<beans>
	<aop:config>
		<!-- 定义一个切入点：myPointcut
			通过expression指定它对应的切入点表达式 -->
		<aop:pointcut id="myPointcut" 
			expression="execution(* org.crazyit.app.service.impl.*.*(..))"/>
		<aop:aspect id="afterThrowingAdviceAspect"
			ref="afterThrowingAdviceBean">
			<!-- 定义一个AfterThrowing增强处理，指定切入点
				以切面Bean中的doRecoveryActions()方法作为增强处理方法 -->
			<aop:after-throwing pointcut-ref="myPointcut" 
				method="doRecoveryActions" throwing="ex"/>
		</aop:aspect>
	</aop:config>
	<!-- 定义一个普通Bean实例，该Bean实例将被作为Aspect Bean -->
	<bean id="afterThrowingAdviceBean"
		class="org.crazyit.app.aspect.RepairAspect"/>
	<bean id="hello" class="org.crazyit.app.service.impl.HelloImpl"/>
	<bean id="world" class="org.crazyit.app.service.impl.WorldImpl"/>
</beans>

```
