---
title: SpringCloudOpenFeign基本使用
date: 2021-11-09 08:12:43
tags: [SpringCloud, Feign]
categories: [SpringCloud, Feign]
---

# Spring Cloud OpenFeign基本使用

该项目通过自动配置和绑定到 Spring Environment 和其他 Spring 编程模型，为 Spring Boot 应用程序提供 OpenFeign 集成。

## 1、声明式 REST 客户端：Feign

Feign 是一个声明式 Web 服务客户端。它使编写 Web 服务客户端变得更容易。要使用 Feign 创建一个接口并对其进行注释。它具有可插入的注释支持，包括 Feign 注释和 JAX-RS 注释。 Feign 还支持可插拔的编码器和解码器。 Spring Cloud 添加了对 Spring MVC 注释的支持，并支持使用 Spring Web 中默认使用的相同 HttpMessageConverters。 Spring Cloud 集成了 Eureka、Spring Cloud CircuitBreaker 和 Spring Cloud LoadBalancer，在使用 Feign 时提供负载均衡的 http 客户端。

### 1.1、如何集成 Feign

要将 Feign 包含在您的项目中，请使用带有group org.springframework.cloud 和artifact ID spring-cloud-starter-openfeign 的 starter。有关使用当前 Spring Cloud Release Train 设置构建系统的详细信息，请参阅 Spring Cloud 项目页面。 

示例：

```java
@SpringBootApplication
@EnableFeignClients
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

**StoreClient.java**

```java
@FeignClient("stores")
public interface StoreClient {
    @RequestMapping(method = RequestMethod.GET, value = "/stores")
    List<Store> getStores();

    @RequestMapping(method = RequestMethod.GET, value = "/stores")
    Page<Store> getStores(Pageable pageable);

    @RequestMapping(method = RequestMethod.POST, value = "/stores/{storeId}", consumes = "application/json")
    Store update(@PathVariable("storeId") Long storeId, Store store);
}
```

在@FeignClient 注释中，String 值（上面的“stores”）是任意客户端名称，用于创建 Spring Cloud LoadBalancer 客户端。您还可以使用 url 属性（绝对值或仅主机名）指定 URL。应用程序上下文中 bean 的名称是接口的完全限定名称。要指定您自己的别名值，您可以使用 @FeignClient 注释的限定符值。

上面的负载平衡器客户端将想要发现“stores”服务的物理地址。如果您的应用程序是 Eureka 客户端，那么它将解析 Eureka 服务注册表中的服务。如果不想使用 Eureka，可以使用 SimpleDiscoveryClient 在外部配置中配置服务器列表。

要在 @Configuration-annotated-classes 上使用 @EnableFeignClients 注释，请确保指定客户端所在的位置，例如：@EnableFeignClients(basePackages = "com.example.clients") 或明确列出它们：@EnableFeignClients(clients = InventoryServiceFeignClient .class）

### 1.2.覆盖 Feign 默认值

Spring Cloud 的 Feign 支持的一个核心概念是命名客户端。每个 feign 客户端都是一个组件集合的一部分，这些组件一起工作以根据需要联系远程服务器，并且集合有一个名称，您可以使用 @FeignClient 注释作为应用程序开发人员为其指定。 Spring Cloud 使用 FeignClientsConfiguration 为每个命名的客户端按需创建一个新的集成作为 ApplicationContext。这包含（除其他外）一个 feign.Decoder、一个 feign.Encoder 和一个 feign.Contract。可以使用 @FeignClient 注释的 contextId 属性来覆盖该集合的名称。

Spring Cloud 允许您通过使用 @FeignClient 声明额外的配置（在 FeignClientsConfiguration 之上）来完全控制 feign 客户端。例子：

```java
@FeignClient(name = "stores", configuration = FooConfiguration.class)
public interface StoreClient {
    //..
}
```

在这种情况下，客户端由 FeignClientsConfiguration 中已有的组件和 FooConfiguration 中的任何组件组成（后者将覆盖前者）。

FooConfiguration 不需要用@Configuration 注释。但是，如果是，那么请注意将其从任何包含此配置的@ComponentScan 中排除，因为在指定时它将成为 feign.Decoder、feign.Encoder、feign.Contract 等的默认源。这可以通过将它放在与任何 @ComponentScan 或 @SpringBootApplication 分开的、不重叠的包中来避免，或者可以在 @ComponentScan 中明确排除它。

除了更改 ApplicationContext 集合的名称之外，使用 @FeignClient 注释的 contextId 属性，它将覆盖客户端名称的别名，并将用作为该客户端创建的配置 bean 名称的一部分。

name 和 url 属性支持占位符。

```java
@FeignClient(name = "${feign.name}", url = "${feign.url}")
public interface StoreClient {
    //..
}
```

Spring Cloud OpenFeign 默认为 feign 提供了以下 bean（BeanType beanName: ClassName）：

- `Decoder`feignDecoder: `ResponseEntityDecoder`( 包装了一个`SpringDecoder`)
- `Encoder` feignEncoder： `SpringEncoder`
- `Logger` feignLogger： `Slf4jLogger`
- `MicrometerCapability`micrometerCapability：如果`feign-micrometer`在类路径上并且`MeterRegistry`可用
- `Contract` feignContract： `SpringMvcContract`
- `Feign.Builder` feignBuilder： `FeignCircuitBreaker.Builder`
- `Client`feignClient：如果 Spring Cloud LoadBalancer 在类路径上，`FeignBlockingLoadBalancerClient`则使用。如果它们都不在类路径上，则使用默认的 feign 客户端。

spring-cloud-starter-openfeign 支持 spring-cloud-starter-loadbalancer。但是，作为一个可选的依赖项，如果您想使用它，您需要确保将其添加到您的项目中。

OkHttpClient 和 ApacheHttpClient 以及 ApacheHC5 feign 客户端可以通过将 feign.okhttp.enabled 或 feign.httpclient.enabled 或 feign.httpclient.hc5.enabled 分别设置为 true 并将它们放在类路径上来使用。您可以通过在使用 Apache 时提供 org.apache.http.impl.client.CloseableHttpClient 或 okhttp3.OkHttpClient 在使用 OK HTTP 或 org.apache.hc.client5.http.impl.classic 时提供 bean 来自定义使用的 HTTP 客户端。使用 Apache HC5 时的 CloseableHttpClient。

Spring Cloud OpenFeign 默认没有为 feign 提供以下 bean，但仍然会从应用程序上下文中查找这些类型的 bean 来创建 feign 客户端：

- `Logger.Level`
- `Retryer`
- `ErrorDecoder`
- `Request.Options`
- `Collection<RequestInterceptor>`
- `SetterFactory`
- `QueryMapEncoder`
- `Capability` (`MicrometerCapability` is provided by default)

默认情况下会创建 Retryer.NEVER_RETRY 类型为 Retryer 的 bean，这将禁用重试。请注意，这种重试行为与 Feign 默认行为不同，它会自动重试 IOExceptions，将它们视为与网络相关的瞬态异常，以及从 ErrorDecoder 抛出的任何 RetryableException。

创建其中一种类型的 bean 并将其放置在 @FeignClient 配置中（例如上面的 FooConfiguration）允许您覆盖所描述的每个 bean。

例子：

```java
@Configuration
public class FooConfiguration {
    @Bean
    public Contract feignContract() {
        return new feign.Contract.Default();
    }

    @Bean
    public BasicAuthRequestInterceptor basicAuthRequestInterceptor() {
        return new BasicAuthRequestInterceptor("user", "password");
    }
}
```

这将 SpringMvcContract 替换为 feign.Contract.Default 并将 RequestInterceptor 添加到 RequestInterceptor 的集合中。

@FeignClient 也可以使用配置属性进行配置。

application.yml

```yaml
feign:
    client:
        config:
            feignName:
                connectTimeout: 5000
                readTimeout: 5000
                loggerLevel: full
                errorDecoder: com.example.SimpleErrorDecoder
                retryer: com.example.SimpleRetryer
                defaultQueryParameters:
                    query: queryValue
                defaultRequestHeaders:
                    header: headerValue
                requestInterceptors:
                    - com.example.FooRequestInterceptor
                    - com.example.BarRequestInterceptor
                decode404: false
                encoder: com.example.SimpleEncoder
                decoder: com.example.SimpleDecoder
                contract: com.example.SimpleContract
                capabilities:
                    - com.example.FooCapability
                    - com.example.BarCapability
                metrics.enabled: false
```

可以以与上述类似的方式在 @EnableFeignClients 属性 defaultConfiguration 中指定默认配置。不同之处在于此配置将适用于所有 feign 客户端。

如果您更喜欢使用配置属性来配置所有 @FeignClient，您可以使用默认的 feign 名称创建配置属性。

您可以使用 feign.client.config.feignName.defaultQueryParameters 和 feign.client.config.feignName.defaultRequestHeaders 来指定将与名为 feignName 的客户端的每个请求一起发送的查询参数和标头。

application.yml

```yaml
feign:
    client:
        config:
            default:
                connectTimeout: 5000
                readTimeout: 5000
                loggerLevel: basic
```

如果我们同时创建@Configuration bean 和配置属性，配置属性将获胜。它将覆盖@Configuration 值。但是如果你想把优先级改成@Configuration，你可以把feign.client.default-to-properties改成false。

如果我们想创建多个具有相同名称或 url 的 feign 客户端，以便它们指向同一服务器但每个具有不同的自定义配置，那么我们必须使用 @FeignClient 的 contextId 属性以避免这些配置的名称冲突Bean。

```java
@FeignClient(contextId = "fooClient", name = "stores", configuration = FooConfiguration.class)
public interface FooClient {
    //..
}
```

```java
@FeignClient(contextId = "barClient", name = "stores", configuration = BarConfiguration.class)
public interface BarClient {
    //..
}
```

也可以将 FeignClient 配置为不从父上下文继承 bean。您可以通过覆盖 FeignClientConfigurer bean 中的 inheritParentConfiguration() 以返回 false 来实现此目的：

```java
@Configuration
public class CustomConfiguration{

@Bean
public FeignClientConfigurer feignClientConfigurer() {
            return new FeignClientConfigurer() {

                @Override
                public boolean inheritParentConfiguration() {
                    return false;
                }
            };

        }
}
```

默认情况下，Feign 客户端不编码斜杠/字符。您可以通过将 feign.client.decodeSlash 的值设置为 false 来更改此行为。

#### 1.2.1. SpringEncoder 配置

在我们提供的 SpringEncoder 中，我们为二进制内容类型设置空字符集，为所有其他类型设置 UTF-8。

您可以修改此行为以通过将 feign.encoder.charset-from-content-type 的值设置为 true 来从 Content-Type 标头字符集派生字符集。

### 1.3.超时处理

我们可以在默认客户端和命名客户端上配置超时。 OpenFeign 使用两个超时参数：

- `connectTimeout` 防止由于服务器处理时间长而阻塞调用者。
- `readTimeout` 从连接建立时开始应用，在返回响应时间过长时触发。

如果服务器未运行或不可用，则数据包会导致连接被拒绝。通信以错误消息或回退结束。如果它设置得非常低，这可能会在 connectTimeout 之前发生。执行查找和接收此类数据包所花费的时间会导致此延迟的很大一部分。它可能会根据涉及 DNS 查找的远程主机进行更改。

### 1.4.手动创建 Feign 客户端

在某些情况下，可能需要以使用上述方法无法实现的方式自定义您的 Feign Client。在这种情况下，您可以使用 Feign Builder API 创建客户端。下面是一个示例，它创建了两个具有相同接口的 Feign 客户端，但为每个客户端配置了一个单独的请求拦截器。

```java
@Import(FeignClientsConfiguration.class)
class FooController {

    private FooClient fooClient;

    private FooClient adminClient;

    @Autowired
    public FooController(Client client, Encoder encoder, Decoder decoder, Contract contract, MicrometerCapability micrometerCapability) {
        this.fooClient = Feign.builder().client(client)
                .encoder(encoder)
                .decoder(decoder)
                .contract(contract)
                .addCapability(micrometerCapability)
                .requestInterceptor(new BasicAuthRequestInterceptor("user", "user"))
                .target(FooClient.class, "https://PROD-SVC");

        this.adminClient = Feign.builder().client(client)
                .encoder(encoder)
                .decoder(decoder)
                .contract(contract)
                .addCapability(micrometerCapability)
                .requestInterceptor(new BasicAuthRequestInterceptor("admin", "admin"))
                .target(FooClient.class, "https://PROD-SVC");
    }
}
```

上面例子中的 FeignClientsConfiguration.class 是 Spring Cloud OpenFeign 提供的默认配置。

PROD-SVC 是客户端将向其发出请求的服务的名称。

Feign Contract 对象定义了接口上哪些注解和值是有效的。自动装配的 Contract bean 提供对 SpringMVC 注释的支持，而不是默认的 Feign 原生注释。

您还可以使用 Builder` 将 FeignClient 配置为不从父上下文继承 bean。您可以通过在 Builder 上覆盖调用 `inheritParentContext(false) 来做到这一点。

### 1.5. Feign Spring Cloud 断路器支持

如果 Spring Cloud CircuitBreaker 在类路径上并且 feign.circuitbreaker.enabled=true，Feign 将使用断路器包装所有方法。 要在每个客户端的基础上禁用 Spring Cloud CircuitBreaker 支持，请创建一个具有“prototype”范围的 vanilla Feign.Builder，例如：

```java
@Configuration
public class FooConfiguration {
    @Bean
    @Scope("prototype")
    public Feign.Builder feignBuilder() {
        return Feign.builder();
    }
}
```

断路器名称遵循此模式 <feignClientClassName>#<CalledMethod>(<parameterTypes>)。当调用带有 FooClient 接口的 @FeignClient 并且被调用的没有参数的接口方法是 bar 时，断路器名称将是 FooClient#bar()。

从 2020.0.2 开始，断路器名称模式已从 <feignClientName>_<CalledMethod> 更改。使用 2020.0.4 中引入的 CircuitBreakerNameResolver，断路器名称可以保留旧模式。

提供 CircuitBreakerNameResolver 的 bean，您可以更改断路器名称模式。

```java
@Configuration
public class FooConfiguration {
    @Bean
    public CircuitBreakerNameResolver circuitBreakerNameResolver() {
        return (String feignClientName, Target<?> target, Method method) -> feignClientName + "_" + method.getName();
    }
}
```

要启用 Spring Cloud CircuitBreaker 组，请将 feign.circuitbreaker.group.enabled 属性设置为 true（默认为 false）。

### 1.6. Feign Spring Cloud 断路器Fallbacks

Spring Cloud CircuitBreaker 支持fallback的概念：当电路打开或出现错误时执行的默认代码路径。要为给定的 @FeignClient 启用回退，请将回退属性设置为实现回退的类名。您还需要将您的实现声明为 Spring bean。

```java
@FeignClient(name = "test", url = "http://localhost:${server.port}/", fallback = Fallback.class)
    protected interface TestClient {

        @RequestMapping(method = RequestMethod.GET, value = "/hello")
        Hello getHello();

        @RequestMapping(method = RequestMethod.GET, value = "/hellonotfound")
        String getException();

    }

    @Component
    static class Fallback implements TestClient {

        @Override
        public Hello getHello() {
            throw new NoFallbackAvailableException("Boom!", new RuntimeException());
        }

        @Override
        public String getException() {
            return "Fixed response";
        }

    }
```

If one needs access to the cause that made the fallback trigger, one can use the `fallbackFactory` attribute inside `@FeignClient`.

```java
@FeignClient(name = "testClientWithFactory", url = "http://localhost:${server.port}/",
            fallbackFactory = TestFallbackFactory.class)
    protected interface TestClientWithFactory {

        @RequestMapping(method = RequestMethod.GET, value = "/hello")
        Hello getHello();

        @RequestMapping(method = RequestMethod.GET, value = "/hellonotfound")
        String getException();

    }

    @Component
    static class TestFallbackFactory implements FallbackFactory<FallbackWithFactory> {

        @Override
        public FallbackWithFactory create(Throwable cause) {
            return new FallbackWithFactory();
        }

    }

    static class FallbackWithFactory implements TestClientWithFactory {

        @Override
        public Hello getHello() {
            throw new NoFallbackAvailableException("Boom!", new RuntimeException());
        }

        @Override
        public String getException() {
            return "Fixed response";
        }

    }
```

### 1.7. Feign 和@Primary

将 Feign 与 Spring Cloud CircuitBreaker fallback一起使用时，ApplicationContext 中有多个相同类型的 bean。这将导致 @Autowired 无法工作，因为没有一个 bean，或者一个标记为主要的 bean。为了解决这个问题，Spring Cloud OpenFeign 将所有 Feign 实例标记为 @Primary，因此 Spring Framework 将知道要注入哪个 bean。在某些情况下，这可能是不可取的。要关闭此行为，请将 @FeignClient 的主要属性设置为 false。

```java
@FeignClient(name = "hello", primary = false)
public interface HelloClient {
    // methods here
}
```

### 1.8. Feign 继承支持

Feign 通过单继承接口支持样板 API。这允许将常见操作分组到方便的基本接口中。

UserService.java

```java
public interface UserService {

    @RequestMapping(method = RequestMethod.GET, value ="/users/{id}")
    User getUser(@PathVariable("id") long id);
}
```

UserResource.java

```java
@RestController
public class UserResource implements UserService {

}
```

UserClient.java

```java
package project.user;

@FeignClient("users")
public interface UserClient extends UserService {

}
```

通常不建议在服务器和客户端之间共享一个接口。它引入了紧耦合，也不是所有维护的 Spring MVC 版本都支持（某些版本没有继承方法参数映射）。

### 1.9. Feign 请求/响应压缩

您可以考虑为您的 Feign 请求启用请求或响应 GZIP 压缩。您可以通过启用以下属性之一来执行此操作：

```java
feign.compression.request.enabled=true
feign.compression.response.enabled=true
```

Feign 请求压缩为您提供类似于您为 Web 服务器设置的设置：

```java
feign.compression.request.enabled=true
feign.compression.request.mime-types=text/xml,application/xml,application/json
feign.compression.request.min-request-size=2048
```

这些属性允许您选择压缩媒体类型和最小请求阈值长度。

对于除了 OkHttpClient 之外的 http 客户端，可以启用默认的 gzip 解码器来解码 UTF-8 编码的 gzip 响应：

```java
feign.compression.response.enabled=true
feign.compression.response.useGzipDecoder=true
```

### 1.10.Feign logging

为每个创建的 Feign 客户端创建一个记录器。默认情况下，记录器的名称是用于创建 Feign 客户端的接口的完整类名。 Feign logging 只响应 DEBUG 级别。

application.yml

```yaml
logging.level.project.user.UserClient: DEBUG
```

您可以为每个客户端配置的 Logger.Level 对象告诉 Feign 要记录多少。选择是：

- `NONE`, 无日志记录（默认）。
- `BASIC`, 仅记录请求方法和 URL 以及响应状态代码和执行时间。
- `HEADERS`, 记录基本信息以及请求和响应标头。
- `FULL`, 记录请求和响应的标头、正文和元数据。

例如，以下内容会将 Logger.Level 设置为 FULL：

```java
@Configuration
public class FooConfiguration {
    @Bean
    Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
}
```

### 1.11. Feign Capability支持

Feign 功能公开了核心 Feign 组件，以便可以修改这些组件。例如，功能可以获取客户端，对其进行装饰，并将装饰后的实例返回给 Feign。对指标库的支持是一个很好的现实例子。请参阅 Feign 指标。

创建一个或多个 Capability bean 并将它们放置在 @FeignClient 配置中，让您可以注册它们并修改相关客户端的行为。

```java
@Configuration
public class FooConfiguration {
    @Bean
    Capability customCapability() {
        return new CustomCapability();
    }
}
```

### 1.12. Feign 指标

如果以下所有条件都为真，则会创建并注册 MicrometerCapability bean，以便您的 Feign 客户端将指标发布到 Micrometer：

- `feign-micrometer` 在classpath上
- A `MeterRegistry` bean可用
- feign 指标属性设置为 true（默认情况下）
  - `feign.metrics.enabled=true` (作用所有客户端)
  - `feign.client.config.feignName.metrics.enabled=true` (作用单个客户端)

如果您的应用程序已经使用 Micrometer，那么启用指标就像将 feign-micrometer 放到您的类路径中一样简单。

您还可以通过以下任一方式禁用该功能：

- 从您的类路径中排除 feign-micrometer
- 将 feign 指标属性之一设置为 false
  - `feign.metrics.enabled=false`
  - `feign.client.config.feignName.metrics.enabled=false`

feign.metrics.enabled=false 禁用对所有 Feign 客户端的度量支持，而不管客户端级别标志的值：feign.client.config.feignName.metrics.enabled。如果要为每个客户端启用或禁用 merics，请不要设置 feign.metrics.enabled 并使用 feign.client.config.feignName.metrics.enabled。

您还可以通过注册自己的 bean 来自定义 MicrometerCapability：

```java
@Configuration
public class FooConfiguration {
    @Bean
    public MicrometerCapability micrometerCapability(MeterRegistry meterRegistry) {
        return new MicrometerCapability(meterRegistry);
    }
}
```

### 1.13. Feign @QueryMap 支持

OpenFeign @QueryMap 注释支持将 POJO 用作 GET 参数映射。不幸的是，默认的 OpenFeign QueryMap 注释与 Spring 不兼容，因为它缺少 value 属性。

Spring Cloud OpenFeign 提供了一个等效的 @SpringQueryMap 注解，用于将 POJO 或 Map 参数注解为查询参数映射。

例如：

Params 类定义了参数 param1 和 param2：

```java
// Params.java
public class Params {
    private String param1;
    private String param2;

    // [Getters and setters omitted for brevity]
}
```

以下 feign 客户端通过使用 @SpringQueryMap 注解来使用 Params 类：

```java
@FeignClient("demo")
public interface DemoTemplate {

    @GetMapping(path = "/demo")
    String demoEndpoint(@SpringQueryMap Params params);
}
```

如果您需要对生成的查询参数映射进行更多控制，则可以实现自定义 QueryMapEncoder bean。

### 1.14. HATEOAS 支持

Spring 提供了一些 API 来创建遵循 HATEOAS 原则、Spring Hateoas 和 Spring Data REST 的 REST 表示。

如果您的项目使用 org.springframework.boot:spring-boot-starter-hateoas starter 或 org.springframework.boot:spring-boot-starter-data-rest starter，则默认启用 Feign HATEOAS 支持。

启用 HATEOAS 支持后，允许 Feign 客户端序列化和反序列化 HATEOAS 表示模型：EntityModel、CollectionModel 和 PagedModel。

```java
@FeignClient("demo")
public interface DemoTemplate {

    @GetMapping(path = "/stores")
    CollectionModel<Store> getStores();
}
```

### 1.15. Spring @MatrixVariable 支持

Spring Cloud OpenFeign 提供对 Spring @MatrixVariable 注解的支持。

如果映射作为方法参数传递，@MatrixVariable 路径段是通过使用 = 连接映射中的键值对来创建的。

如果传递了不同的对象，则@MatrixVariable 批注（如果已定义）中提供的名称或带批注的变量名称使用 = 与提供的方法参数连接。

即使在服务器端，Spring 不要求用户将路径段占位符命名为与matrix variable名称相同的名称，因为它在客户端过于模糊，Spring Cloud OpenFeign 要求您添加一个路径段占位符与@MatrixVariable 注释（如果已定义）中提供的名称或带注释的变量名称匹配的名称。

例如：

```java
@GetMapping("/objects/links/{matrixVars}")
Map<String, List<String>> getObjects(@MatrixVariable Map<String, List<String>> matrixVars);
```

请注意，变量名称和路径段占位符都称为 matrixVars。

```java
@FeignClient("demo")
public interface DemoTemplate {

    @GetMapping(path = "/stores")
    CollectionModel<Store> getStores();
}
```

### 1.16. Feign CollectionFormat 支持

我们通过提供 @CollectionFormat 注释来支持 feign.CollectionFormat。您可以通过传递所需的 feign.CollectionFormat 作为注释值来注释 Feign 客户端方法。

在以下示例中，使用 CSV 格式而不是默认的 EXPLODED 来处理方法。

```java
@FeignClient(name = "demo")
    protected interface PageableFeignClient {

        @CollectionFormat(feign.CollectionFormat.CSV)
        @GetMapping(path = "/page")
        ResponseEntity performRequest(Pageable page);

    }
```

在发送 Pageable 作为查询参数时设置 CSV 格式，以便正确编码。

1.17.Reactive支持

由于 OpenFeign 项目目前不支持响应式客户端，例如 Spring WebClient，Spring Cloud OpenFeign 也不支持。我们将在核心项目中尽快添加对它的支持。

在完成之前，我们建议使用 feign-reactive 来支持 Spring WebClient。

#### 1.17.1.早期初始化错误

根据您使用 Feign 客户端的方式，您可能会在启动应用程序时看到初始化错误。要解决此问题，您可以在自动装配客户端时使用 ObjectProvider。

```java
@Autowired
ObjectProvider<TestFeginClient> testFeginClient;
```

### 1.18.Spring Data支持

您可以考虑启用 Jackson Modules 以支持 org.springframework.data.domain.Page 和 org.springframework.data.domain.Sort 解码。

```java
feign.autoconfiguration.jackson.enabled=true
```

### 1.19. Spring @RefreshScope 支持

如果启用了 Feign 客户端刷新，则使用 feign.Request.Options 作为刷新范围的 bean 创建每个 feign 客户端。这意味着可以通过 POST /actuator/refresh 针对任何 Feign 客户端实例刷新诸如 connectTimeout 和 readTimeout 之类的属性。

默认情况下，Feign 客户端中的刷新行为是禁用的。使用以下属性启用刷新行为：

```java
feign.client.refresh-enabled=true
```

不要用@RefreshScope 注释来注释@FeignClient 接口。

## 2、配置属性

可以在 application.properties 文件、application.yml 文件或命令行开关中指定各种属性。本附录提供了常见 Spring Cloud OpenFeign 属性的列表以及对使用它们的底层类的引用。

属性贡献可以来自类路径上的其他 jar 文件，因此您不应认为这是一个详尽的列表。此外，您可以定义自己的属性。

| 名称                                         | 默认值                                          | 描述                                                         |
| :------------------------------------------- | :---------------------------------------------- | :----------------------------------------------------------- |
| feign.autoconfiguration.jackson.enabled      | `false`                                         | 如果为 true，将为 Jackson 页面解码提供 PageJacksonModule 和 SortJacksonModule bean。 |
| feign.circuitbreaker.enabled                 | `false`                                         | 如果为 true，则 OpenFeign 客户端将使用 Spring Cloud CircuitBreaker 断路器包装。 |
| feign.circuitbreaker.group.enabled           | `false`                                         | 如果为 true，则 OpenFeign 客户端将使用带有组的 Spring Cloud CircuitBreaker 断路器包装。 |
| feign.client.config                          |                                                 |                                                              |
| feign.client.decode-slash                    | `true`                                          | Feign 客户端默认不编码斜杠/字符。要更改此行为，请将 decodeSlash 设置为 false。 |
| feign.client.default-config                  | `default`                                       |                                                              |
| feign.client.default-to-properties           | `true`                                          |                                                              |
| feign.client.refresh-enabled                 | `false`                                         | 为 Feign 启用选项值刷新功能。                                |
| feign.compression.request.enabled            | `false`                                         | 使 Feign 发送的请求能够被压缩。                              |
| feign.compression.request.mime-types         | `[text/xml, application/xml, application/json]` | 支持的 MIME 类型列表。                                       |
| feign.compression.request.min-request-size   | `2048`                                          | 最小阈值内容大小。                                           |
| feign.compression.response.enabled           | `false`                                         | 使来自 Feign 的响应能够被压缩。                              |
| feign.compression.response.useGzipDecoder    | `false`                                         | 启用要使用的默认 gzip 解码器。                               |
| feign.encoder.charset-from-content-type      | `false`                                         | 指示字符集是否应从 {@code Content-Type} 标头派生。           |
| feign.httpclient.connection-timeout          | `2000`                                          |                                                              |
| feign.httpclient.connection-timer-repeat     | `3000`                                          |                                                              |
| feign.httpclient.disable-ssl-validation      | `false`                                         |                                                              |
| feign.httpclient.enabled                     | `true`                                          | 允许 Feign 使用 Apache HTTP 客户端。                         |
| feign.httpclient.follow-redirects            | `true`                                          |                                                              |
| feign.httpclient.hc5.enabled                 | `false`                                         | 允许 Feign 使用 Apache HTTP Client 5。                       |
| feign.httpclient.hc5.pool-concurrency-policy |                                                 | 池并发策略。                                                 |
| feign.httpclient.hc5.pool-reuse-policy       |                                                 | 池连接重用策略。                                             |
| feign.httpclient.hc5.socket-timeout          | `5`                                             | Socket超时的默认值。                                         |
| feign.httpclient.hc5.socket-timeout-unit     |                                                 | Socket超时单位的默认值。                                     |
| feign.httpclient.max-connections             | `200`                                           |                                                              |
| feign.httpclient.max-connections-per-route   | `50`                                            |                                                              |
| feign.httpclient.time-to-live                | `900`                                           |                                                              |
| feign.httpclient.time-to-live-unit           |                                                 |                                                              |
| feign.metrics.enabled                        | `true`                                          | 为 Feign 启用指标功能。                                      |
| feign.okhttp.enabled                         | `false`                                         | 允许 Feign 使用 OK HTTP Client。                             |