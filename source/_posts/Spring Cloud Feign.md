---
title: Spring Cloud Feign
tags: 
    - SpringCloud
    - 学习笔记
categories:  
    - [JAVA后端开发, SpringCloud]
date: 2022-06-25 00:00:00
---
# Spring Cloud Feign

## Feign概述

​	Feign是一个声明式的Web Service客户端，是一种声明式、模板化的HTTP客户端。在Spring Cloud中使用Feign，可以做到使用HTTP请求访问远程服务，就像是调用本地方法一样的，开发者完全感知不到这是在调用远程方法，更感知不到在访问HTTP请求。

Feign特性：

1. 可拔插的注解支持，包括Feign注解和JAX-RS注解
2. 支持可拔插的HTTP编码器和解码器
3. 支持Hystrix和它的Fallback
4. 支持HTTP请求和响应的压缩



## Feign工作原理

- 在开发微服务应用时，我们会在主程序入口添加@EnableFeignClinents注解开启对Feign Client扫描加载处理。根据Feign Client的开发规范，定义接口并加上@FeignClient注解
- 当程序启动时，会进行包扫描，扫描所有@FeignClient的注解的类，并将这些信息注入Spring IOC容器中。当定义的Feign接口中的方法被调用时，通过JDK的代理方式，来生成具体的RequestTemplate。当生成代理时，Feign会为每个接口方法创建一个RequestTemplate对象，该对象封装了HTTP请求需要的全部信息，如请求参数名、请求方法等信息都是在这个过程中确定的
- 然后有RequestTemplate生成Request，然后把Request交给Client去处理，这是指的Client可以是JDK原生的URLConnection、Apache的Http Client，也可以是Okhttp。最后Client被封装到LoadBalanceClient类，这个类结合Ribbon负载均衡发起服务之间的调用

## Feign的基础功能

### FeignClient注解

FeignClient注解被@Target(ElementType.TYPE)修饰，表示FeignClient注解多种目标在接口上。常用属性如下：

- name：指定FeignClient的名称，如果项目使用了Ribbon，name属性会作为微服务的名称，用于服务发现
- url：url一般用于调试，可以受动指定@FeignClient嗲用的地址
- decode404：当放生404错误时，如果该字段为true，会调用decoder进行解码，否则抛出FeignException异常
- configuration：Feign配置类，可以定义Feign的Encoder、Decoder、LogLevel、Contract
- fallback：定义容错的处理类，当调用远程接口失败或超时时，会调用对应接口的容错逻辑，fallback指定的类必须实现@FeignClient标记的接口
- fallbackFactory：工厂类，用于生成fallback类实例，通过这个属性我们可以实现每个接口通用的容错逻辑，减少重复的代码
- path：定义当前FeignClient的统一前缀

### Feign开启GZIP压缩

``` yaml
feign:
	compression:
		request:
			enabled: true
			mimd-type: text/xml,application/xml,application/json # 配置压缩支持的MIME类型
			min-request-size: 2048
		response:
			enabled: true #配置响应GZIP压缩
```

开启GZIP压缩之后，Feign之间的调用通过二进制协议进行传输，返回值需要修改为`ResponseEntity<byte[]>`才可以正常显示，否则会导致服务之间的调用结果乱码

### Feign属性配置

1. 对单个指定特定名称的Feign进行配置

   ```yaml
   feign:
   	client:
   		config:
   			feignName: #需要配置的FeignName
   				connectTimout: 5000	#连接超时时间
   				readTimeout: 5000 #读超时时间设置
   				loggerLevel: full #配置Feign的日子级别
   				errorDcoder: com.example.SimpleErrorDecoder #Feign的错误解码器
   				retryer: com.example.SimpleRetryer #配置重试
   				requestInterceptors: #配置拦截器
   					- com.example.FooRequestInterceptor 
   					- com.example.BarRequestInterceptor
   				decode404: false 
   				encoder: com.example.SimpleEncoder #Feign的编码器
   				decoder: com.example.SimpleDecoder #Feign的解码器
   				contrace: com.example.SimpleContract #Feign的Contract配置
   ```

2. 作用于所有Feign的配置方式

   java配置方式：

   ```java
   @SpringBootApplication
   @EnableFeignClients(defaultConfiguration=DefaultFeignConfiguration.class)
   public class ConsumerApplication{
       public static void main(String[] args){
           SpringApplication.run(ConsumerApplication.class, args);
       }
   }
   ```

   配置文件方式：

   ```yaml
   feign:
   	client:
   		config:
   			default:
   				connectTimeout: 5000
   				readTimeout: 5000
   				loggerLevel: basic
   ```

   

### Feign Client开启日志

1. 在application.yml中设置日志输出级别
2. 通过java代码的方式在主程序入口类中配置日志Bean，或者在@Configuration配置中去配置

### Feign的超时配置

Feign的调用分两层，即Ribbon的调用和Hystrix的调用，高版本的Hystrix默认是关闭的。

```yaml
ribbon.ReadTimeout: 120000
ribbon.ConnectTimeout : 30000

feign.hystrix.enabled: true
hystrix:
	shareSecurityContext: true
	command:
		default:
			circuitBreaker:
				sleepWindowInMilliseconds: 100000
				forceClosed: true
			execution:
				isolation:
					thread:
						timeoutInMilliseconds: 600000
```

### Feign默认Client的替换

Feign在默认情况下使用的是JDK原生的URLConnection发送HTTP请求，没有连接池，但是对每个地址会保持一个长连接，即利用HTTP的persistence connection。可以使用Apache的HTTP Client替换Feign原始的HTTP Client，通过设置连接池、超时时间等对服务之间的调用调优

1. 使用HTTP Client替换Feign默认Client

   ```xml
   <dependencies>
       <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-web</artifactId>
       </dependency>
       <!-- Spring Cloud OpenFeign的Starter的依赖 -->
       <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-openfeign</artifactId>
       </dependency>
   
       <!-- 使用Apache HttpClient替换Feign原生httpclient -->
       <dependency>
       <groupId>org.apache.httpcomponents</groupId>
       <artifactId>httpclient</artifactId>
       </dependency>
   
       <dependency>
       <groupId>com.netflix.feign</groupId>
       <artifactId>feign-httpclient</artifactId>
       <version>8.17.0</version>
       </dependency>
   </dependencies>
   ```

   ```yaml
   server:
     port: 8010
   spring:
     application:
       name: ch4-3-httpclient
   
   feign:
     httpclient:
         enabled: true
   ```

   

2. 使用okhttp替换Feign默认Client

   ```xml
   <dependencies>
       <dependency>
           <groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-starter-web</artifactId>
       </dependency>
       <!-- Spring Cloud OpenFeign的Starter的依赖 -->
       <dependency>
           <groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-starter-openfeign</artifactId>
       </dependency>
       <dependency>
           <groupId>io.github.openfeign</groupId>
           <artifactId>feign-okhttp</artifactId>
       </dependency>
   </dependencies>
   ```

   ```yaml
   server:
     port: 8011
   spring:
     application:
       name: ch4-3-okhttp
   
   
   feign:
       httpclient:
            enabled: false
       okhttp:
            enabled: true
   ```

   ```java
   @Configuration
   @ConditionalOnClass(Feign.class)
   @AutoConfigureBefore(FeignAutoConfiguration.class)
   public class FeignOkHttpConfig {
       @Bean
       public okhttp3.OkHttpClient okHttpClient(){
           return new okhttp3.OkHttpClient.Builder()
                    //设置连接超时
                   .connectTimeout(60, TimeUnit.SECONDS)
                   //设置读超时
                   .readTimeout(60, TimeUnit.SECONDS)
                   //设置写超时
                   .writeTimeout(60,TimeUnit.SECONDS)
                   //是否自动重连
                   .retryOnConnectionFailure(true)
                   .connectionPool(new ConnectionPool())
                   //构建OkHttpClient对象
                   .build();
       }
   
   }
   ```

   

### Feign的Post和Get的多参数传递

通过实现Feign的RequestInterceptor中的apply方法来进行统一拦截转换处理Feign中的GET方法多参数传递房问题。

``` java
@Component
public class FeignRequestInterceptor implements RequestInterceptor {

    @Autowired
    private ObjectMapper objectMapper;

    @Override
    public void apply(RequestTemplate template) {
        // feign 不支持 GET 方法传 POJO, json body转query
        if (template.method().equals("GET") && template.body() != null) {
            try {
                JsonNode jsonNode = objectMapper.readTree(template.body());
                template.body(null);

                Map<String, Collection<String>> queries = new HashMap<>();
                buildQuery(jsonNode, "", queries);
                template.queries(queries);
            } catch (IOException e) {
                //提示:根据实践项目情况处理此处异常，这里不做扩展。
                e.printStackTrace();
            }
        }
    }

    private void buildQuery(JsonNode jsonNode, String path, Map<String, Collection<String>> queries) {
        if (!jsonNode.isContainerNode()) {   // 叶子节点
            if (jsonNode.isNull()) {
                return;
            }
            Collection<String> values = queries.get(path);
            if (null == values) {
                values = new ArrayList<>();
                queries.put(path, values);
            }
            values.add(jsonNode.asText());
            return;
        }
        if (jsonNode.isArray()) {   // 数组节点
            Iterator<JsonNode> it = jsonNode.elements();
            while (it.hasNext()) {
                buildQuery(it.next(), path, queries);
            }
        } else {
            Iterator<Map.Entry<String, JsonNode>> it = jsonNode.fields();
            while (it.hasNext()) {
                Map.Entry<String, JsonNode> entry = it.next();
                if (StringUtils.hasText(path)) {
                    buildQuery(entry.getValue(), path + "." + entry.getKey(), queries);
                } else {  // 根节点
                    buildQuery(entry.getValue(), entry.getKey(), queries);
                }
            }
        }
    }
}

```

### Feign调用传递Toke

在进行认证鉴权时，当使用Feign时就会发现外部请求到A服务的时候，A服务是可以拿到Token的，然而当服务使用Feign调用B服务的时候，Token就会丢失，从而认证失败。需要做的就是在Feign调用的时候，向请求头俩面添加需要传递的Token

```java
@Component
public class FeignTokenInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate requestTemplate) {
        if(null==getHttpServletRequest()){
            //此处省略日志记录
            return;
        }
        //将获取Token对应的值往下面传
        requestTemplate.header("oauthToken", getHeaders(getHttpServletRequest()).get("oauthToken"));
    }

    private HttpServletRequest getHttpServletRequest() {
        try {
            return ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest();
        } catch (Exception e) {
            return null;
        }
    }

    /**
     * Feign拦截器拦截请求获取Token对应的值
     * @param request
     * @return
     */
    private Map<String, String> getHeaders(HttpServletRequest request) {
        Map<String, String> map = new LinkedHashMap<>();
        Enumeration<String> enumeration = request.getHeaderNames();
        while (enumeration.hasMoreElements()) {
            String key = enumeration.nextElement();
            String value = request.getHeader(key);
            map.put(key, value);
        }
        return map;
    }
}
```

### 解决Feign首次请求失败问题

当Feign和Ribbon整合了Hystrix之后，可能出现首次调用失败的问题：

​	Hystrix默认的超时时间时1S，如果超过这个时间未做出响应，将会进入fallback代码。由于Bean的装配以及懒加载机制等，Feign首次请求会比较慢。如果这个响应时间大于1S，就会出现请求失败的问题。

1. 法一：将Hystrix的超时时间改为5S `hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds: 5000`
2. 法二：禁用Hystrix的超时时间 `hystrix.command.default.execution.timeout.enabled: false`
3. 法三：使用Feign的时候直接关闭Hystrix（不推荐）`feign.hystrix.enabled.false`

### Feign返回图片流处理方式

通过Feign返回图片一般为字节数组，但因为Controller层的返回值不能直接返回byte，因此需要将Feign的返回值修改为response

### Feign的文件上传

可以通过Feign官方提供的feign-form，其中实现了上传需要的Encoder

```xml
<!-- Feign文件上传依赖-->
<dependency>
    <groupId>io.github.openfeign.form</groupId>
    <artifactId>feign-form</artifactId>
    <version>3.0.3</version>
</dependency>

<dependency>
    <groupId>io.github.openfeign.form</groupId>
    <artifactId>feign-form-spring</artifactId>
    <version>3.0.3</version>
</dependency>

```

```java
@FeignClient(value = "feign-file-server", configuration = FeignMultipartSupportConfig.class)
public interface FileUploadFeignService {

    /***
     * 1.produces,consumes必填
     * 2.注意区分@RequestPart和RequestParam，不要将
     * @RequestPart(value = "file") 写成@RequestParam(value = "file")
     * @param file
     * @return
     */
    @RequestMapping(method = RequestMethod.POST, value = "/uploadFile/server",
            produces = {MediaType.APPLICATION_JSON_UTF8_VALUE},
            consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public  String fileUpload(@RequestPart(value = "file") MultipartFile file);

}
```

```java
@Configuration
public class FeignMultipartSupportConfig {

    @Bean
    @Primary
    @Scope("prototype")
    public Encoder multipartFormEncoder() {
        return new SpringFormEncoder();
    }
}
```



### venus-cloud-feign的使用

1. 解决Spring MVC Controller中的方法不支持继承实现Feign接口中方法参数上的注解问题
2. Feign不支持GET方法传递POJO的问题

