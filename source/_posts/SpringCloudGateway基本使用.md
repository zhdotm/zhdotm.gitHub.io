---
title: SpringCloudGateway基本使用
date: 2021-11-09 08:12:43
tags: [SpringCloud, Gateway]
categories: [SpringCloud, Gateway]
--- 

# Spring Cloud Gateway基本使用

本项目提供了一个构建在 Spring 生态系统之上的 API 网关，包括：Spring 5、Spring Boot 2 和 Project Reactor。 Spring Cloud Gateway 旨在提供一种简单而有效的方式来路由到 API 并为它们提供交叉关注点，例如：安全性、监控/指标和弹性。

## 1.如何集成Spring Cloud Gateway

要将 Spring Cloud Gateway 包含在您的项目中，请使用具有 org.springframework.cloud 的组 ID 和 spring-cloud-starter-gateway 的工件 ID 的 starter。有关使用当前 Spring Cloud Release Train 设置构建系统的详细信息，请参阅 Spring Cloud 项目页面。

如果包含启动器，但不希望启用网关，请设置 spring.cloud.gateway.enabled=false。

Spring Cloud Gateway 基于 Spring Boot 2.x、Spring WebFlux 和 Project Reactor 构建。因此，当您使用 Spring Cloud Gateway 时，您所知道的许多熟悉的同步库（例如 Spring Data 和 Spring Security）和模式可能不适用。如果您不熟悉这些项目，我们建议您在使用 Spring Cloud Gateway 之前先阅读他们的文档以熟悉一些新概念。

Spring Cloud Gateway 需要 Spring Boot 和 Spring Webflux 提供的 Netty 运行时。它不适用于传统的 Servlet 容器或构建为 WAR 时。

## 2. 词汇表

- **Route**: 网关的基本构建块。它由 ID、目标 URI、谓词集合和过滤器集合定义。如果聚合谓词为真，则匹配路由。
- **Predicate**: 这是一个 Java 8 函数谓词。输入类型是 Spring Framework ServerWebExchange。这使您可以匹配来自 HTTP 请求的任何内容，例如标头或参数。
- **Filter**: 这些是使用特定工厂构建的 GatewayFilter 实例。在这里，您可以在发送下游请求之前或之后修改请求和响应。

## 3. 工作原理

下图提供了 Spring Cloud Gateway 工作原理的高级概述：

![Spring Cloud Gateway Diagram](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/images/spring_cloud_gateway_diagram.png)

客户端向 Spring Cloud Gateway 发出请求。如果网关处理程序映射确定请求与路由匹配，则将其发送到网关 Web 处理程序。此处理程序通过特定于请求的过滤器链运行请求。过滤器被虚线分隔的原因是过滤器可以在发送代理请求之前和之后运行逻辑。执行所有“预”过滤器逻辑。然后进行代理请求。发出代理请求后，将运行“post”过滤器逻辑。

在没有端口的路由中定义的 URI 分别获得 HTTP 和 HTTPS URI 的默认端口值 80 和 443。

## 4. 配置路由谓词工厂和网关过滤工厂

有两种方法可以配置谓词和过滤器：快捷方式和完全扩展的参数。下面的大多数示例都使用快捷方式。

名称和参数名称将在每个部分的第一句或第二句中作为代码列出。参数通常按快捷方式配置所需的顺序列出。

### 4.1.快捷方式配置

快捷方式配置由过滤器名称识别，后跟等号 (=)，后跟由逗号 (,) 分隔的参数值。

**application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: after_route
        uri: https://example.org
        predicates:
        - Cookie=mycookie,mycookievalue
```

前面的示例使用两个参数定义了 Cookie 路由谓词工厂，即 cookie 名称、mycookie 和匹配 mycookievalue 的值。

### 4.2.完全展开的参数

完全扩展的参数看起来更像是带有名称/值对的标准 yaml 配置。通常，会有一个 name 键和一个 args 键。 args 键是键值对的映射，用于配置谓词或过滤器。

**application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: after_route
        uri: https://example.org
        predicates:
        - name: Cookie
          args:
            name: mycookie
            regexp: mycookievalue
```

这就是上面显示的 Cookie 谓词的快捷配置的完整配置。

## 5. 路由谓词工厂

Spring Cloud Gateway 匹配路由作为 Spring WebFlux HandlerMapping 基础结构的一部分。 Spring Cloud Gateway 包含许多内置的路由谓词工厂。所有这些谓词都匹配 HTTP 请求的不同属性。您可以将多个路由谓词工厂与逻辑和语句组合在一起。

### 5.1.after路由谓词工厂

After 路由谓词工厂接受一个参数，一个日期时间（这是一个 java ZonedDateTime）。此谓词匹配在指定日期时间之后发生的请求。以下示例配置了一个 after 路由谓词：

**Example 1. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: after_route
        uri: https://example.org
        predicates:
        - After=2017-01-20T17:42:47.789-07:00[America/Denver]
```

此路由匹配Jan 20, 2017 17:42 Mountain Time (Denver)之后提出的任何请求。

### 5.2.before路由谓词工厂

Before 路由谓词工厂接受一个参数，一个日期时间（它是一个 java ZonedDateTime）。此谓词匹配在指定日期时间之前发生的请求。以下示例配置了一个 before 路由谓词：

**Example 2. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: before_route
        uri: https://example.org
        predicates:
        - Before=2017-01-20T17:42:47.789-07:00[America/Denver]
```

此路由匹配  Jan 20, 2017 17:42 Mountain Time (Denver)之前提出的任何请求。

### 5.3.between路由谓词工厂

路由谓词工厂之间有两个参数，datetime1 和 datetime2，它们是 java ZonedDateTime 对象。此谓词匹配发生在 datetime1 之后和 datetime2 之前的请求。 datetime2 参数必须在 datetime1 之后。以下示例配置了一个 between 路由谓词：

**Example 3. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: between_route
        uri: https://example.org
        predicates:
        - Between=2017-01-20T17:42:47.789-07:00[America/Denver], 2017-01-21T17:42:47.789-07:00[America/Denver]
```

此路由匹配 2017-01-20T17:42:47.789-07:00[America/Denver]之后和 2017-01-21T17:42:47.789-07:00[America/Denver]之前提出的任何请求。这对于维护窗口可能很有用。

### 5.4. Cookie 路由谓词工厂

Cookie 路由谓词工厂有两个参数，即 cookie 名称和一个 regexp（这是一个 Java 正则表达式）。此谓词匹配具有给定名称且其值与正则表达式匹配的 cookie。以下示例配置 cookie 路由谓词工厂：

**Example 4. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: cookie_route
        uri: https://example.org
        predicates:
        - Cookie=chocolate, ch.p
```

此路由匹配具有名为 Chocolate 的 cookie 的请求，该 cookie 的值与 ch.p 正则表达式匹配。

### 5.5.header路由谓词工厂

Header 路由谓词工厂接受两个参数，标题名称和一个 regexp（这是一个 Java 正则表达式）。此谓词与具有给定名称的标头匹配，其值与正则表达式匹配。以下示例配置标头路由谓词：

**Example 5. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: header_route
        uri: https://example.org
        predicates:
        - Header=X-Request-Id, \d+
```

如果请求具有名为 X-Request-Id 的标头，其值与 \d+ 正则表达式匹配（即，它具有一个或多个数字的值），则此路由匹配。

### 5.6.host路由谓词工厂

Host路由谓词工厂采用一个参数：host名模式列表。该模式是 Ant 风格的模式，带有 .作为分隔符。此谓词匹配与模式匹配的 Host 标头。以下示例配置host路由谓词：

**Example 6. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: host_route
        uri: https://example.org
        predicates:
        - Host=**.somehost.org,**.anotherhost.org
```

还支持 URI 模板变量（例如 {sub}.myhost.org）。

如果请求具有值为 www.somehost.org 或 beta.somehost.org 或 www.anotherhost.org 的 Host 标头，则此路由匹配。

此谓词提取 URI 模板变量（例如，在前面的示例中定义的 sub）作为名称和值的映射，并将其放置在 ServerWebExchange.getAttributes() 中，并使用在 ServerWebExchangeUtils.URI_TEMPLATE_VARIABLES_ATTRIBUTE 中定义的键。然后这些值可供 GatewayFilter 工厂使用

### 5.7.method路由谓词工厂

方法路由谓词工厂采用一个方法参数，它是一个或多个参数：要匹配的 HTTP 方法。以下示例配置方法路由谓词：

**Example 7. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: method_route
        uri: https://example.org
        predicates:
        - Method=GET,POST
```

如果请求方法是 GET 或 POST，则此路由匹配。

### 5.8.path路由谓词工厂

Path Route Predicate Factory 接受两个参数：一个 Spring PathMatcher 模式列表和一个名为 matchTrailingSlash 的可选标志（默认为 true）。以下示例配置路径路由谓词：

**Example 8. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: path_route
        uri: https://example.org
        predicates:
        - Path=/red/{segment},/blue/{segment}
```

如果请求路径是例如：/red/1 或 /red/1/ 或 /red/blue 或 /blue/green，则此路由匹配。

如果 matchTrailingSlash 设置为 false，则不会匹配请求路径 /red/1/。

此谓词提取 URI 模板变量（例如在前面的示例中定义的段）作为名称和值的映射，并将其放置在 ServerWebExchange.getAttributes() 中，键是在 ServerWebExchangeUtils.URI_TEMPLATE_VARIABLES_ATTRIBUTE 中定义的。然后这些值可供 GatewayFilter 工厂使用 可以使用实用方法（称为 get）来更轻松地访问这些变量。以下示例显示了如何使用 get 方法：

```java
Map<String, String> uriVariables = ServerWebExchangeUtils.getPathPredicateVariables(exchange);

String segment = uriVariables.get("segment");
```

### 5.9.query路由谓词工厂

Query路由谓词工厂有两个参数：一个必需的参数和一个可选的正则表达式（它是一个 Java 正则表达式）。以下示例配置查询路由谓词：

**Example 9. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: query_route
        uri: https://example.org
        predicates:
        - Query=green
```

如果请求包含绿色查询参数，则前面的路由匹配。

**application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: query_route
        uri: https://example.org
        predicates:
        - Query=red, gree.
```

如果请求包含值与 gree 匹配的红色查询参数，则前面的路由匹配。 regexp，所以 green 和 greet 会匹配。

### 5.10. RemoteAddr 路由谓词工厂

RemoteAddr 路由谓词工厂采用源列表（最小大小 1），这些源是 CIDR 表示法（IPv4 或 IPv6）字符串，例如 192.168.0.1/16（其中 192.168.0.1 是 IP 地址，16 是子网掩码）。以下示例配置 RemoteAddr 路由谓词：

**Example 10. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: remoteaddr_route
        uri: https://example.org
        predicates:
        - RemoteAddr=192.168.1.1/24
```

如果请求的远程地址是例如 192.168.1.10，则此路由匹配。

### 5.11.Weight权重路由谓词工厂

Weight 路由谓词工厂采用两个参数：group 和 weight（一个 int）。权重按组计算。以下示例配置权重路由谓词：

**Example 11. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: weight_high
        uri: https://weighthigh.org
        predicates:
        - Weight=group1, 8
      - id: weight_low
        uri: https://weightlow.org
        predicates:
        - Weight=group1, 2
```

该路由会将约 80% 的流量转发到 weighthigh.org，将约 20% 的流量转发到 weightlow.org

#### 5.11.1.修改远程地址的解析方式

默认情况下，RemoteAddr 路由谓词工厂使用来自传入请求的远程地址。如果 Spring Cloud Gateway 位于代理层之后，这可能与实际客户端 IP 地址不匹配。

您可以通过设置自定义 RemoteAddressResolver 来自定义解析远程地址的方式。 Spring Cloud Gateway 带有一个基于 X-Forwarded-For 标头 XForwardedRemoteAddressResolver 的非默认远程地址解析器。

XForwardedRemoteAddressResolver 有两个静态构造函数方法，它们采取不同的安全方法：

- XForwardedRemoteAddressResolver::trustAll 返回一个 RemoteAddressResolver，它总是采用在 X-Forwarded-For 标头中找到的第一个 IP 地址。这种方法容易受到欺骗，因为恶意客户端可以为 X-Forwarded-For 设置一个初始值，该值将被解析器接受。
- XForwardedRemoteAddressResolver::maxTrustedIndex 采用一个索引，该索引与运行在 Spring Cloud Gateway 前面的受信任基础设施的数量相关。例如，如果 Spring Cloud Gateway 只能通过 HAProxy 访问，则应使用值 1。如果在访问 Spring Cloud Gateway 之前需要两跳可信基础设施，则应使用值 2。

考虑以下标头值：

```yaml
X-Forwarded-For: 0.0.0.1, 0.0.0.2, 0.0.0.3
```

以下 maxTrustedIndex 值产生以下远程地址：

| `maxTrustedIndex`        | 结果                                          |
| :----------------------- | :-------------------------------------------- |
| [`Integer.MIN_VALUE`,0]  | （无效，初始化期间 IllegalArgumentException） |
| 1                        | 0.0.0.3                                       |
| 2                        | 0.0.0.2                                       |
| 3                        | 0.0.0.1                                       |
| [4, `Integer.MAX_VALUE`] | 0.0.0.1                                       |

以下示例显示了如何使用 Java 实现相同的配置：

Example 12. GatewayConfig.java

```java
RemoteAddressResolver resolver = XForwardedRemoteAddressResolver
    .maxTrustedIndex(1);

...

.route("direct-route",
    r -> r.remoteAddr("10.1.1.1", "10.10.1.1/24")
        .uri("https://downstream1")
.route("proxied-route",
    r -> r.remoteAddr(resolver, "10.10.1.1", "10.10.1.1/24")
        .uri("https://downstream2")
)
```

## 6. GatewayFilter 工厂

路由过滤器允许以某种方式修改传入的 HTTP 请求或传出的 HTTP 响应。路由过滤器的范围是特定的路由。 Spring Cloud Gateway 包含许多内置的 GatewayFilter 工厂。

### 6.1. AddRequestHeader 网关过滤器工厂

AddRequestHeader GatewayFilter 工厂采用名称和值参数。以下示例配置 AddRequestHeader GatewayFilter：

**Example 13. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: add_request_header_route
        uri: https://example.org
        filters:
        - AddRequestHeader=X-Request-red, blue
```

此清单将 X-Request-red:blue 标头添加到所有匹配请求的下游请求标头中。

AddRequestHeader 知道用于匹配路径或主机的 URI 变量。 URI 变量可以在值中使用并在运行时扩展。以下示例配置使用变量的 AddRequestHeader GatewayFilter：

**Example 14. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: add_request_header_route
        uri: https://example.org
        predicates:
        - Path=/red/{segment}
        filters:
        - AddRequestHeader=X-Request-Red, Blue-{segment}
```

### 6.2. AddRequestParameter 网关过滤器工厂

AddRequestParameter GatewayFilter Factory 采用名称和值参数。以下示例配置 AddRequestParameter GatewayFilter：

**Example 15. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: add_request_parameter_route
        uri: https://example.org
        filters:
        - AddRequestParameter=red, blue
```

这会将 red=blue 添加到所有匹配请求的下游请求的查询字符串中。

AddRequestParameter 知道用于匹配路径或主机的 URI 变量。 URI 变量可以在值中使用并在运行时扩展。以下示例配置使用变量的 AddRequestParameter GatewayFilter：

**Example 16. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: add_request_parameter_route
        uri: https://example.org
        predicates:
        - Host: {segment}.myhost.org
        filters:
        - AddRequestParameter=foo, bar-{segment}
```

### 6.3. AddResponseHeader 网关过滤器工厂

AddResponseHeader GatewayFilter Factory 采用名称和值参数。以下示例配置 AddResponseHeader GatewayFilter：

**Example 17. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: add_response_header_route
        uri: https://example.org
        filters:
        - AddResponseHeader=X-Response-Red, Blue
```

这会将 X-Response-Foo:Bar 标头添加到所有匹配请求的下游响应标头中。

AddResponseHeader 知道用于匹配路径或主机的 URI 变量。 URI 变量可以在值中使用并在运行时扩展。以下示例配置使用变量的 AddResponseHeader GatewayFilter：

**Example 18. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: add_response_header_route
        uri: https://example.org
        predicates:
        - Host: {segment}.myhost.org
        filters:
        - AddResponseHeader=foo, bar-{segment}
```

### 6.4. DedupeResponseHeader 网关过滤器工厂

DedupeResponseHeader GatewayFilter 工厂采用名称参数和可选的策略参数。 name 可以包含以空格分隔的标题名称列表。以下示例配置 DedupeResponseHeader GatewayFilter：

**Example 19. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: dedupe_response_header_route
        uri: https://example.org
        filters:
        - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
```

在网关 CORS 逻辑和下游逻辑都添加它们的情况下，这会删除 Access-Control-Allow-Credentials 和 Access-Control-Allow-Origin 响应标头的重复值。

DedupeResponseHeader 过滤器还接受一个可选的策略参数。接受的值为 RETAIN_FIRST（默认）、RETAIN_LAST 和 RETAIN_UNIQUE。

### 6.5. Spring Cloud 断路器网关过滤器工厂

Spring Cloud CircuitBreaker GatewayFilter 工厂使用 Spring Cloud CircuitBreaker API 将网关路由包装在断路器中。 Spring Cloud CircuitBreaker 支持多个可与 Spring Cloud Gateway 一起使用的库。 Spring Cloud 支持开箱即用的 Resilience4J。

要启用 Spring Cloud CircuitBreaker 过滤器，您需要将 spring-cloud-starter-circuitbreaker-reactor-resilience4j 放在类路径上。以下示例配置 Spring Cloud CircuitBreaker GatewayFilter：

**Example 20. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: circuitbreaker_route
        uri: https://example.org
        filters:
        - CircuitBreaker=myCircuitBreaker
```

要配置断路器，请参阅您正在使用的底层断路器实现的配置。

- [Resilience4J Documentation](https://cloud.spring.io/spring-cloud-circuitbreaker/reference/html/spring-cloud-circuitbreaker.html)

Spring Cloud CircuitBreaker 过滤器还可以接受可选的 fallbackUri 参数。目前，只支持转发：schemed URIs。如果调用回退，则请求将转发到与 URI 匹配的控制器。以下示例配置了这样的回退：

**Example 21. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: circuitbreaker_route
        uri: lb://backing-service:8088
        predicates:
        - Path=/consumingServiceEndpoint
        filters:
        - name: CircuitBreaker
          args:
            name: myCircuitBreaker
            fallbackUri: forward:/inCaseOfFailureUseThis
        - RewritePath=/consumingServiceEndpoint, /backingServiceEndpoint
```

下面的清单在 Java 中做了同样的事情：

**Example 22. Application.java**

```java
@Bean
public RouteLocator routes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("circuitbreaker_route", r -> r.path("/consumingServiceEndpoint")
            .filters(f -> f.circuitBreaker(c -> c.name("myCircuitBreaker").fallbackUri("forward:/inCaseOfFailureUseThis"))
                .rewritePath("/consumingServiceEndpoint", "/backingServiceEndpoint")).uri("lb://backing-service:8088")
        .build();
}
```

当调用断路器回退时，此示例转发到 /inCaseofFailureUseThis URI。请注意，此示例还演示了（可选）Spring Cloud LoadBalancer 负载平衡（由目标 URI 上的 lb 前缀定义）。

主要场景是使用 fallbackUri 在网关应用程序中定义内部控制器或处理程序。但是，您也可以将请求重新路由到外部应用程序中的控制器或处理程序，如下所示：

**Example 23. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: ingredients
        uri: lb://ingredients
        predicates:
        - Path=//ingredients/**
        filters:
        - name: CircuitBreaker
          args:
            name: fetchIngredients
            fallbackUri: forward:/fallback
      - id: ingredients-fallback
        uri: http://localhost:9994
        predicates:
        - Path=/fallback
```

在此示例中，网关应用程序中没有回退端点或处理程序。但是，在另一个应用程序中有一个，在 localhost:9994 下注册。

如果请求被转发到回退，Spring Cloud CircuitBreaker Gateway 过滤器还提供导致它的 Throwable。它作为 ServerWebExchangeUtils.CIRCUITBREAKER_EXECUTION_EXCEPTION_ATTR 属性添加到 ServerWebExchange 中，可在网关应用程序中处理回退时使用。

对于外部控制器/处理程序场景，可以添加带有异常详细信息的标头。您可以在 FallbackHeaders GatewayFilter Factory 部分找到有关这样做的更多信息。

#### 6.5.1.根据状态代码使断路器脱扣

在某些情况下，您可能希望根据从其环绕的路由返回的状态代码来触发断路器。断路器配置对象采用状态代码列表，如果返回这些状态代码，将导致断路器跳闸。在设置要使断路器跳闸的状态代码时，您可以使用带有状态代码值的整数或 HttpStatus 枚举的字符串表示形式。

**Example 24. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: circuitbreaker_route
        uri: lb://backing-service:8088
        predicates:
        - Path=/consumingServiceEndpoint
        filters:
        - name: CircuitBreaker
          args:
            name: myCircuitBreaker
            fallbackUri: forward:/inCaseOfFailureUseThis
            statusCodes:
              - 500
              - "NOT_FOUND"
```

**Example 25. Application.java**

```java
@Bean
public RouteLocator routes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("circuitbreaker_route", r -> r.path("/consumingServiceEndpoint")
            .filters(f -> f.circuitBreaker(c -> c.name("myCircuitBreaker").fallbackUri("forward:/inCaseOfFailureUseThis").addStatusCode("INTERNAL_SERVER_ERROR"))
                .rewritePath("/consumingServiceEndpoint", "/backingServiceEndpoint")).uri("lb://backing-service:8088")
        .build();
}
```

6.6. FallbackHeaders 网关过滤器工厂

FallbackHeaders 工厂允许您在转发到外部应用程序中的 fallbackUri 的请求的标头中添加 Spring Cloud CircuitBreaker 执行异常详细信息，如下面的场景：

**Example 26. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: ingredients
        uri: lb://ingredients
        predicates:
        - Path=//ingredients/**
        filters:
        - name: CircuitBreaker
          args:
            name: fetchIngredients
            fallbackUri: forward:/fallback
      - id: ingredients-fallback
        uri: http://localhost:9994
        predicates:
        - Path=/fallback
        filters:
        - name: FallbackHeaders
          args:
            executionExceptionTypeHeaderName: Test-Header
```

### 6.6. FallbackHeaders 网关过滤器工厂

FallbackHeaders 工厂允许您在转发到外部应用程序中的 fallbackUri 的请求的标头中添加 Spring Cloud CircuitBreaker 执行异常详细信息，如下面的场景：

**Example 26. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: ingredients
        uri: lb://ingredients
        predicates:
        - Path=//ingredients/**
        filters:
        - name: CircuitBreaker
          args:
            name: fetchIngredients
            fallbackUri: forward:/fallback
      - id: ingredients-fallback
        uri: http://localhost:9994
        predicates:
        - Path=/fallback
        filters:
        - name: FallbackHeaders
          args:
            executionExceptionTypeHeaderName: Test-Header
```

在此示例中，在运行断路器时发生执行异常后，请求将转发到在 localhost:9994 上运行的应用程序中的回退端点或处理程序。 FallbackHeaders 过滤器将带有异常类型、消息和（如果可用）根本原因异常类型和消息的标头添加到该请求中。

您可以通过设置以下参数的值（显示为它们的默认值）来覆盖配置中标头的名称：

- `executionExceptionTypeHeaderName` (`"Execution-Exception-Type"`)
- `executionExceptionMessageHeaderName` (`"Execution-Exception-Message"`)
- `rootCauseExceptionTypeHeaderName` (`"Root-Cause-Exception-Type"`)
- `rootCauseExceptionMessageHeaderName` (`"Root-Cause-Exception-Message"`)

有关断路器和网关的更多信息，请参阅 Spring Cloud CircuitBreaker Factory 部分。

### 6.7. MapRequestHeader 网关过滤器工厂

MapRequestHeader GatewayFilter 工厂采用 fromHeader 和 toHeader 参数。它创建一个新的命名标头 (toHeader)，并从传入的 http 请求中从现有命名标头 (fromHeader) 中提取该值。如果输入标头不存在，则过滤器没有影响。如果新命名的标头已存在，则其值将使用新值进行扩充。以下示例配置 MapRequestHeader：

**Example 27. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: map_request_header_route
        uri: https://example.org
        filters:
        - MapRequestHeader=Blue, X-Request-Red
```

这会将 X-Request-Red:<values> 标头添加到下游请求中，并使用来自传入 HTTP 请求的 Blue 标头的更新值。

### 6.8. PrefixPath 网关过滤器工厂

PrefixPath GatewayFilter 工厂采用单个前缀参数。以下示例配置 PrefixPath GatewayFilter：

**Example 28. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: prefixpath_route
        uri: https://example.org
        filters:
        - PrefixPath=/mypath
```

这会将 /mypath 前缀为所有匹配请求的路径。因此，对 /hello 的请求将被发送到 /mypath/hello。

### 6.9. PreserveHostHeader 网关过滤器工厂

PreserveHostHeader GatewayFilter 工厂没有参数。此过滤器设置路由过滤器检查的请求属性，以确定是否应发送原始主机标头，而不是由 HTTP 客户端确定的主机标头。以下示例配置 PreserveHostHeader GatewayFilter：

**Example 29. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: preserve_host_route
        uri: https://example.org
        filters:
        - PreserveHostHeader
```

### 6.10. RequestRateLimiter 网关过滤器工厂

RequestRateLimiter GatewayFilter 工厂使用 RateLimiter 实现来确定是否允许继续处理当前请求。如果不是，则返回 HTTP 429 - Too Many Requests（默认情况下）状态。

此过滤器采用可选的 keyResolver 参数和特定于速率限制器的参数（本节稍后介绍）。

keyResolver 是一个实现 KeyResolver 接口的 bean。在配置中，使用 SpEL 按名称引用 bean。 #{@myKeyResolver} 是一个 SpEL 表达式，它引用名为 myKeyResolver 的 bean。以下清单显示了 KeyResolver 接口：

**Example 30. KeyResolver.java**

```java
public interface KeyResolver {
    Mono<String> resolve(ServerWebExchange exchange);
}
```

KeyResolver 接口让可插拔策略派生出限制请求的密钥。在未来的里程碑版本中，将有一些 KeyResolver 实现。

KeyResolver 的默认实现是 PrincipalNameKeyResolver，它从 ServerWebExchange 检索 Principal 并调用 Principal.getName()。

默认情况下，如果 KeyResolver 未找到密钥，则拒绝请求。您可以通过设置 spring.cloud.gateway.filter.request-rate-limiter.deny-empty-key（true 或 false）和 spring.cloud.gateway.filter.request-rate-limiter.empty-key 来调整此行为-状态码属性。

RequestRateLimiter 不能使用“快捷方式”表示法进行配置。以下示例无效：

**Example 31. application.properties**

```properties
# INVALID SHORTCUT CONFIGURATION
spring.cloud.gateway.routes[0].filters[0]=RequestRateLimiter=2, 2, #{@userkeyresolver}
```

#### 6.10.1. The Redis RateLimiter

Redis 实现基于在 Stripe 完成的工作。它需要使用 spring-boot-starter-data-redis-reactive Spring Boot starter。

使用的算法是令牌桶算法。

redis-rate-limiter.replenishRate 属性是您希望允许用户每秒执行多少请求，而没有任何丢弃的请求。这是令牌桶填充的速率。

redis-rate-limiter.burstCapacity 属性是允许用户在一秒内执行的最大请求数。这是令牌桶可以容纳的令牌数量。将此值设置为零会阻止所有请求。

redis-rate-limiter.requestedTokens 属性是请求花费多少令牌。这是每个请求从存储桶中获取的令牌数量，默认为 1。

稳定速率是通过在replyRate 和burstCapacity 中设置相同的值来实现的。通过将burstCapacity 设置为高于replyRate，可以允许临时突发。在这种情况下，需要允许速率限制器在突发之间有一段时间（根据replyRate），因为两个连续的突发将导致请求丢失（HTTP 429 - Too Many Requests）。以下清单配置了 redis-rate-limiter：

低于 1 请求/秒的速率限制是通过将replyRate 设置为所需的请求数量、将requestedTokens 设置为以秒为单位的时间跨度以及将burstCapacity 设置为replyRate 和requestedTokens 的乘积来实现的，例如设置replyRate=1、requestedTokens=60 和burstCapacity=60 将导致1 请求/分钟的限制。

**Example 32. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: requestratelimiter_route
        uri: https://example.org
        filters:
        - name: RequestRateLimiter
          args:
            redis-rate-limiter.replenishRate: 10
            redis-rate-limiter.burstCapacity: 20
            redis-rate-limiter.requestedTokens: 1
```

以下示例在 Java 中配置 KeyResolver：

**Example 33. Config.java**

```java
@Bean
KeyResolver userKeyResolver() {
    return exchange -> Mono.just(exchange.getRequest().getQueryParams().getFirst("user"));
}
```

这定义了每个用户 10 的请求速率限制。允许突发 20 个，但在下一秒，只有 10 个请求可用。 KeyResolver 是一个简单的获取用户请求参数的方法（注意，不推荐用于生产）。

您还可以将速率限制器定义为实现 RateLimiter 接口的 bean。在配置中，您可以使用 SpEL 按名称引用 bean。 #{@myRateLimiter} 是一个 SpEL 表达式，它引用名为 myRateLimiter 的 bean。下面的清单定义了一个速率限制器，它使用在前面的清单中定义的 KeyResolver：

**Example 34. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: requestratelimiter_route
        uri: https://example.org
        filters:
        - name: RequestRateLimiter
          args:
            rate-limiter: "#{@myRateLimiter}"
            key-resolver: "#{@userKeyResolver}"
```

### 6.11. RedirectTo 网关过滤器工厂

RedirectTo GatewayFilter 工厂接受两个参数，status 和 url。 status 参数应该是一个 300 系列的重定向 HTTP 代码，比如 301。url 参数应该是一个有效的 URL。这是 Location 标头的值。对于相对重定向，您应该使用 uri: no://op 作为路由定义的 uri。以下清单配置了一个 RedirectTo GatewayFilter：

**Example 35. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: prefixpath_route
        uri: https://example.org
        filters:
        - RedirectTo=302, https://acme.org
```

这将发送带有 Location:https://acme.org 标头的状态 302 以执行重定向。

### 6.12. RemoveRequestHeader 网关过滤器工厂

RemoveRequestHeader GatewayFilter 工厂采用 name 参数。它是要删除的标题的名称。以下清单配置了 RemoveRequestHeader GatewayFilter：

**Example 36. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: removerequestheader_route
        uri: https://example.org
        filters:
        - RemoveRequestHeader=X-Request-Foo
```

这会在向下游发送之前删除 X-Request-Foo 标头。

### 6.13. RemoveResponseHeader 网关过滤器工厂

RemoveResponseHeader GatewayFilter 工厂采用 name 参数。它是要删除的标题的名称。以下清单配置了 RemoveResponseHeader GatewayFilter：

**Example 37. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: removeresponseheader_route
        uri: https://example.org
        filters:
        - RemoveResponseHeader=X-Response-Foo
```

这将在响应返回到网关客户端之前从响应中删除 X-Response-Foo 标头。

要删除任何类型的敏感标头，您应该为您可能想要这样做的任何路由配置此过滤器。此外，您可以使用 spring.cloud.gateway.default-filters 配置一次此过滤器，并将其应用于所有路由。

### 6.14. RemoveRequestParameter 网关过滤器工厂

RemoveRequestParameter GatewayFilter 工厂采用名称参数。它是要删除的查询参数的名称。以下示例配置 RemoveRequestParameter GatewayFilter：

**Example 38. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: removerequestparameter_route
        uri: https://example.org
        filters:
        - RemoveRequestParameter=red
```

这将在向下游发送之前删除red参数。

### 6.15. RewritePath 网关过滤器工厂

RewritePath GatewayFilter 工厂采用路径正则表达式参数和替换参数。这使用 Java 正则表达式来灵活地重写请求路径。以下清单配置了 RewritePath GatewayFilter：

**Example 39. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: rewritepath_route
        uri: https://example.org
        predicates:
        - Path=/red/**
        filters:
        - RewritePath=/red/?(?<segment>.*), /$\{segment}
```

对于 /red/blue 的请求路径，这会在发出下游请求之前将路径设置为 /blue。请注意，由于 YAML 规范，$ 应替换为 $\。

### 6.16. RewriteLocationResponseHeader 网关过滤器工厂

RewriteLocationResponseHeader GatewayFilter 工厂修改 Location 响应头的值，通常是为了摆脱后端特定的细节。它采用 stripVersionMode、locationHeaderName、hostValue 和 protocolsRegex 参数。以下清单配置了 RewriteLocationResponseHeader GatewayFilter：

**Example 40. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: rewritelocationresponseheader_route
        uri: http://example.org
        filters:
        - RewriteLocationResponseHeader=AS_IN_REQUEST, Location, ,
```

例如，对于POST api.example.com/some/object/name的请求，object-service.prod.example.net/v2/some/object/id的Location响应头值改写为api.example.com/some/object/id。

stripVersionMode 参数具有以下可能的值：NEVER_STRIP、AS_IN_REQUEST（默认）和 ALWAYS_STRIP。

- `NEVER_STRIP`: 即使原始请求路径不包含版本，也不会剥离版本。
- `AS_IN_REQUEST` 仅当原始请求路径不包含版本时才会剥离版本。
- `ALWAYS_STRIP` 版本总是被剥离，即使原始请求路径包含版本。

hostValue 参数（如果提供）用于替换响应 Location 标头的 host:port 部分。如果未提供，则使用 Host 请求标头的值。

protocolRegex 参数必须是有效的正则表达式字符串，与协议名称匹配。如果不匹配，则过滤器不执行任何操作。默认为 http|https|ftp|ftps。

### 6.17. RewriteResponseHeader 网关过滤器工厂

RewriteResponseHeader GatewayFilter 工厂采用名称、正则表达式和替换参数。它使用 Java 正则表达式来灵活地重写响应头值。以下示例配置 RewriteResponseHeader GatewayFilter：

**Example 41. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: rewriteresponseheader_route
        uri: https://example.org
        filters:
        - RewriteResponseHeader=X-Response-Red, , password=[^&]+, password=***
```

对于 /42?user=ford&password=omg!what&flag=true 的 header 值，在发出下游请求后设置为 /42?user=ford&password=***&flag=true。由于 YAML 规范，您必须使用 $\ 来表示 $。

### 6.18. SaveSession 网关过滤器工厂

SaveSession GatewayFilter 工厂在向下游转发调用之前强制执行 WebSession::save 操作。这在将 Spring Session 之类的东西与惰性数据存储一起使用时特别有用，并且您需要确保在进行转发调用之前已保存会话状态。以下示例配置 SaveSession GatewayFilter：

**Example 42. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: save_session
        uri: https://example.org
        predicates:
        - Path=/foo/**
        filters:
        - SaveSession
```

如果您将 Spring Security 与 Spring Session 集成并希望确保安全详细信息已转发到远程进程，那么这很关键。

### 6.19. SecureHeaders 网关过滤器工厂

根据本博客文章中提出的建议，SecureHeaders GatewayFilter 工厂向响应添加了许多标头。

添加了以下header（显示为默认值）：

- `X-Xss-Protection:1 (mode=block`)
- `Strict-Transport-Security (max-age=631138519`)
- `X-Frame-Options (DENY)`
- `X-Content-Type-Options (nosniff)`
- `Referrer-Policy (no-referrer)`
- `Content-Security-Policy (default-src 'self' https:; font-src 'self' https: data:; img-src 'self' https: data:; object-src 'none'; script-src https:; style-src 'self' https: 'unsafe-inline)'`
- `X-Download-Options (noopen)`
- `X-Permitted-Cross-Domain-Policies (none)`

要更改默认值，请在 spring.cloud.gateway.filter.secure-headers 命名空间中设置适当的属性。以下属性可用：

- `xss-protection-header`
- `strict-transport-security`
- `x-frame-options`
- `x-content-type-options`
- `referrer-policy`
- `content-security-policy`
- `x-download-options`
- `x-permitted-cross-domain-policies`

要禁用默认值，请使用逗号分隔值设置 spring.cloud.gateway.filter.secure-headers.disable 属性。以下示例显示了如何执行此操作：

```properties
spring.cloud.gateway.filter.secure-headers.disable=x-frame-options,strict-transport-security
```

### 6.20. SetPath 网关过滤器工厂

SetPath GatewayFilter 工厂采用路径模板参数。它提供了一种通过允许路径的模板化段来操作请求路径的简单方法。这使用了 Spring Framework 中的 URI 模板。允许多个匹配段。以下示例配置 SetPath GatewayFilter：

**Example 43. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: setpath_route
        uri: https://example.org
        predicates:
        - Path=/red/{segment}
        filters:
        - SetPath=/{segment}
```

对于 /red/blue 的请求路径，这会在发出下游请求之前将路径设置为 /blue。

### 6.21. SetRequestHeader 网关过滤器工厂

SetRequestHeader GatewayFilter 工厂采用名称和值参数。以下清单配置了 SetRequestHeader GatewayFilter：

**Example 44. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: setrequestheader_route
        uri: https://example.org
        filters:
        - SetRequestHeader=X-Request-Red, Blue
```

此 GatewayFilter 替换（而不是添加）具有给定名称的所有标头。因此，如果下游服务器以 X-Request-Red:1234 响应，这将替换为 X-Request-Red:Blue，这是下游服务将收到的。

SetRequestHeader 知道用于匹配路径或主机的 URI 变量。 URI 变量可以在值中使用并在运行时扩展。以下示例配置了一个使用变量的 SetRequestHeader GatewayFilter：

**Example 45. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: setrequestheader_route
        uri: https://example.org
        predicates:
        - Host: {segment}.myhost.org
        filters:
        - SetRequestHeader=foo, bar-{segment}
```

### 6.22. SetResponseHeader 网关过滤器工厂

SetResponseHeader GatewayFilter 工厂采用名称和值参数。以下清单配置了 SetResponseHeader GatewayFilter：

**Example 46. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: setresponseheader_route
        uri: https://example.org
        filters:
        - SetResponseHeader=X-Response-Red, Blue
```

此 GatewayFilter 替换（而不是添加）具有给定名称的所有标头。因此，如果下游服务器以 X-Response-Red:1234 响应，这将替换为 X-Response-Red:Blue，这是网关客户端将收到的。

SetResponseHeader 知道用于匹配路径或主机的 URI 变量。 URI 变量可以在值中使用，并将在运行时扩展。以下示例配置使用变量的 SetResponseHeader GatewayFilter：

**Example 47. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: setresponseheader_route
        uri: https://example.org
        predicates:
        - Host: {segment}.myhost.org
        filters:
        - SetResponseHeader=foo, bar-{segment}
```

### 6.23. SetStatus 网关过滤器工厂

SetStatus GatewayFilter 工厂采用单个参数 status。它必须是有效的 Spring HttpStatus。它可能是整数值 404 或枚举的字符串表示形式：NOT_FOUND。以下清单配置了 SetStatus GatewayFilter：

**Example 48. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: setstatusstring_route
        uri: https://example.org
        filters:
        - SetStatus=BAD_REQUEST
      - id: setstatusint_route
        uri: https://example.org
        filters:
        - SetStatus=401
```

无论哪种情况，响应的 HTTP 状态都设置为 401。

您可以配置 SetStatus GatewayFilter 以在响应的标头中返回来自代理请求的原始 HTTP 状态代码。如果配置了以下属性，则将标头添加到响应中：

**Example 49. application.yml**

```yaml
spring:
  cloud:
    gateway:
      set-status:
        original-status-header-name: original-http-status
```

### 6.24. StripPrefix 网关过滤器工厂

StripPrefix GatewayFilter 工厂采用一个参数，parts。部分参数指示在将请求发送到下游之前要从请求中剥离的路径中的部分数。以下清单配置了 StripPrefix GatewayFilter：

**Example 50. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: nameRoot
        uri: https://nameservice
        predicates:
        - Path=/name/**
        filters:
        - StripPrefix=2
```

当通过网关向 /name/blue/red 发出请求时，对 nameservice 发出的请求看起来像 nameservice/red。

### 6.25. Retry网关过滤器工厂

Retry GatewayFilter 工厂支持以下参数：

- `retries`: 应该尝试的重试次数。
- `statuses`: 应该重试的 HTTP 状态码，用 org.springframework.http.HttpStatus 表示。
- `methods`: 应该重试的 HTTP 方法，使用 org.springframework.http.HttpMethod 表示。
- `series`: 要重试的一系列状态码，用 org.springframework.http.HttpStatus.Series 表示。
- `exceptions`: 应该重试的抛出异常的列表。
- `backoff`: 为重试配置的指数backoff。在 firstBackoff * (factor ^ n) 的backoff间隔后执行重试，其中 n 是迭代。如果配置了 maxBackoff，则应用的最大backoff限制为 maxBackoff。如果 basedOnPreviousValue 为真，则使用 prevBackoff * factor计算回退。

如果启用，则为重试过滤器配置以下默认值：

- `retries`: 三次
- `series`: 5XX系列
- `methods`: GET 方法
- `exceptions`: IOException 和 TimeoutException
- `backoff`: disabled

以下清单配置了重试网关过滤器：

**Example 51. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: retry_test
        uri: http://localhost:8080/flakey
        predicates:
        - Host=*.retry.com
        filters:
        - name: Retry
          args:
            retries: 3
            statuses: BAD_GATEWAY
            methods: GET,POST
            backoff:
              firstBackoff: 10ms
              maxBackoff: 50ms
              factor: 2
              basedOnPreviousValue: false
```

使用带有 forward: 前缀的重试过滤器时，应仔细编写目标端点，以便在出现错误时不会执行任何可能导致响应被发送到客户端并提交的操作。例如，如果目标端点是带注释的控制器，则目标控制器方法不应返回带有错误状态代码的 ResponseEntity。相反，它应该抛出异常或发出错误信号（例如，通过 Mono.error(ex) 返回值），重试过滤器可以配置为通过重试来处理。

将重试过滤器与任何带有正文的 HTTP 方法一起使用时，正文将被缓存并且网关将受到内存限制。正文缓存在由 ServerWebExchangeUtils.CACHED_REQUEST_BODY_ATTR 定义的请求属性中。对象的类型是 org.springframework.core.io.buffer.DataBuffer。

### 6.26. RequestSize 网关过滤器工厂

当请求大小大于允许的限制时，RequestSize GatewayFilter 工厂可以限制请求到达下游服务。过滤器采用 maxSize 参数。 maxSize 是一个 `DataSize 类型，因此值可以定义为一个数字，后跟一个可选的 DataUnit 后缀，例如 'KB' 或 'MB'。字节的默认值为“B”。它是以字节为单位定义的请求的允许大小限制。以下清单配置了 RequestSize GatewayFilter：

**Example 52. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: request_size_route
        uri: http://localhost:8080/upload
        predicates:
        - Path=/upload
        filters:
        - name: RequestSize
          args:
            maxSize: 5000000
```

当请求因大小而被拒绝时，RequestSize GatewayFilter 工厂将响应状态设置为 413 Payload Too Large 并带有一个额外的标头 errorMessage。以下示例显示了这样的错误消息：

```
errorMessage` : `Request size is larger than permissible limit. Request size is 6.0 MB where permissible limit is 5.0 MB
```

如果未在路由定义中作为过滤器参数提供，则默认请求大小设置为 5 MB。

### 6.27. SetRequestHostHeader 网关过滤器工厂

在某些情况下，可能需要覆盖主机标头。在这种情况下， SetRequestHostHeader GatewayFilter 工厂可以用指定的值替换现有的主机头。过滤器采用主机参数。以下清单配置了 SetRequestHostHeader GatewayFilter：

**Example 53. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: set_request_host_header_route
        uri: http://localhost:8080/headers
        predicates:
        - Path=/headers
        filters:
        - name: SetRequestHostHeader
          args:
            host: example.org
```

SetRequestHostHeader GatewayFilter 工厂用 example.org 替换主机标头的值。

### 6.28.修改请求正文 GatewayFilter Factory

您可以使用 ModifyRequestBody 过滤器过滤器在请求正文被网关发送到下游之前对其进行修改。

此过滤器只能通过使用 Java DSL 进行配置。

以下清单显示了如何修改请求正文 GatewayFilter：

```java
@Bean
public RouteLocator routes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("rewrite_request_obj", r -> r.host("*.rewriterequestobj.org")
            .filters(f -> f.prefixPath("/httpbin")
                .modifyRequestBody(String.class, Hello.class, MediaType.APPLICATION_JSON_VALUE,
                    (exchange, s) -> return Mono.just(new Hello(s.toUpperCase())))).uri(uri))
        .build();
}

static class Hello {
    String message;

    public Hello() { }

    public Hello(String message) {
        this.message = message;
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }
}
```

如果请求没有正文，则 RewriteFilter 将传递 null。应返回 Mono.empty() 以在请求中分配缺失的主体。

### 6.29.修改响应体 GatewayFilter 工厂

您可以使用 ModifyResponseBody 过滤器在响应正文发送回客户端之前对其进行修改。

此过滤器只能通过使用 Java DSL 进行配置。

以下清单显示了如何修改响应正文 GatewayFilter：

```java
@Bean
public RouteLocator routes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("rewrite_response_upper", r -> r.host("*.rewriteresponseupper.org")
            .filters(f -> f.prefixPath("/httpbin")
                .modifyResponseBody(String.class, String.class,
                    (exchange, s) -> Mono.just(s.toUpperCase()))).uri(uri))
        .build();
}
```

如果响应没有正文，则 RewriteFilter 将传递 null。应返回 Mono.empty() 以在响应中分配缺失的主体。

### 6.30.令牌Relay网关过滤器工厂

令牌中继是 OAuth2 消费者充当客户端并将传入令牌转发到传出资源请求的地方。消费者可以是纯客户端（如 SSO 应用程序）或资源服务器。

Spring Cloud Gateway 可以将 OAuth2 访问令牌下游转发到它正在代理的服务。要将此功能添加到网关，您需要像这样添加 TokenRelayGatewayFilterFactory：

**App.java**

```java
@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
    return builder.routes()
            .route("resource", r -> r.path("/resource")
                    .filters(f -> f.tokenRelay())
                    .uri("http://localhost:9000"))
            .build();
}
```

或这个

**application.yaml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: resource
        uri: http://localhost:9000
        predicates:
        - Path=/resource
        filters:
        - TokenRelay=
```

并且它将（除了登录用户并获取令牌之外）将身份验证令牌下游传递给服务（在本例中为 /resource）。

要为 Spring Cloud Gateway 启用此功能，请添加以下依赖项

- `org.springframework.boot:spring-boot-starter-oauth2-client`

它是如何工作的？ {githubmaster}/src/main/java/org/springframework/cloud/gateway/security/TokenRelayGatewayFilterFactory.java[filter] 从当前已验证的用户中提取访问令牌，并将其放入下游请求的请求头中。

只有在设置了正确的 spring.security.oauth2.client.* 属性时才会创建 TokenRelayGatewayFilterFactory bean，这将触发 ReactiveClientRegistrationRepository bean 的创建。

TokenRelayGatewayFilterFactory 使用的 ReactiveOAuth2AuthorizedClientService 的默认实现使用内存数据存储。如果您需要更强大的解决方案，您将需要提供自己的实现 ReactiveOAuth2AuthorizedClientService。

### 6.31.默认过滤器

要添加过滤器并将其应用于所有路由，您可以使用 spring.cloud.gateway.default-filters。此属性采用过滤器列表。以下清单定义了一组默认过滤器：

**Example 54. application.yml**

```yaml
spring:
  cloud:
    gateway:
      default-filters:
      - AddResponseHeader=X-Response-Default-Red, Default-Blue
      - PrefixPath=/httpbin
```

## 7. 全局过滤器

GlobalFilter 接口与 GatewayFilter 具有相同的签名。这些是有条件地应用于所有路由的特殊过滤器。

在未来的里程碑版本中，此界面及其用法可能会发生变化。

### 7.1.组合全局过滤器和网关过滤器排序

当请求与路由匹配时，过滤 Web 处理程序会将 GlobalFilter 的所有实例和 GatewayFilter 的所有特定于路由的实例添加到过滤器链中。这个组合过滤器链由 org.springframework.core.Ordered 接口排序，您可以通过实现 getOrder() 方法设置该接口。

由于 Spring Cloud Gateway 区分过滤器逻辑执行的“pre”和“post”阶段，具有最高优先级的过滤器是“pre”阶段的第一个，“post”阶段的最后一个——阶段。

以下清单配置了过滤器链：

**Example 55. ExampleConfiguration.java**

```java
@Bean
public GlobalFilter customFilter() {
    return new CustomGlobalFilter();
}

public class CustomGlobalFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        log.info("custom global filter");
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return -1;
    }
}
```

### 7.2.Forward路由过滤器

ForwardRoutingFilter 在交换属性 ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR 中查找 URI。如果 URL 有转发方案（例如 forward:///localendpoint），它会使用 Spring DispatcherHandler 来处理请求。请求 URL 的路径部分被转发 URL 中的路径覆盖。未修改的原始 URL 将附加到 ServerWebExchangeUtils.GATEWAY_ORIGINAL_REQUEST_URL_ATTR 属性中的列表中。

### 7.3. ReactiveLoadBalancerClientFilter

ReactiveLoadBalancerClientFilter 在名为 ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR 的交换属性中查找 URI。如果 URL 具有 lb 方案（例如 lb://myservice），则它使用 Spring Cloud ReactorLoadBalancer 将名称（在此示例中为 myservice）解析为实际主机和端口，并替换同一属性中的 URI。未修改的原始 URL 将附加到 ServerWebExchangeUtils.GATEWAY_ORIGINAL_REQUEST_URL_ATTR 属性中的列表中。过滤器还会查看 ServerWebExchangeUtils.GATEWAY_SCHEME_PREFIX_ATTR 属性以查看它是否等于 lb。如果是，则应用相同的规则。以下清单配置了一个 ReactiveLoadBalancerClientFilter：

**Example 56. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: myRoute
        uri: lb://service
        predicates:
        - Path=/service/**
```

默认情况下，当 ReactorLoadBalancer 找不到服务实例时，会返回 503。您可以通过设置 spring.cloud.gateway.loadbalancer.use404=true 将网关配置为返回 404。

从 ReactiveLoadBalancerClientFilter 返回的 ServiceInstance 的 isSecure 值会覆盖向网关发出的请求中指定的方案。例如，如果请求通过 HTTPS 进入网关，但 ServiceInstance 指示它不安全，则通过 HTTP 发出下游请求。相反的情况也可以适用。但是，如果在网关配置中为路由指定了 GATEWAY_SCHEME_PREFIX_ATTR，则前缀将被剥离，并且来自路由 URL 的结果方案将覆盖 ServiceInstance 配置。

### 7.4. Netty 路由过滤器

如果位于 ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR 交换属性中的 URL 具有 http 或 https 方案，则 Netty 路由过滤器运行。它使用 Netty HttpClient 发出下游代理请求。响应放在 ServerWebExchangeUtils.CLIENT_RESPONSE_ATTR 交换属性中，以供稍后过滤器使用。 （还有一个实验性的 WebClientHttpRoutingFilter 执行相同的功能但不需要 Netty。）

### 7.5. Netty 写响应过滤器

如果 ServerWebExchangeUtils.CLIENT_RESPONSE_ATTR 交换属性中存在 Netty HttpClientResponse，则 NettyWriteResponseFilter 运行。它在所有其他过滤器完成后运行，并将代理响应写回网关客户端响应。 （还有一个实验性的 WebClientWriteResponseFilter 可以执行相同的功能，但不需要 Netty。）

### 7.6. RouteToRequestUrl 过滤器

如果 ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR 交换属性中存在 Route 对象，则 RouteToRequestUrlFilter 运行。它基于请求 URI 创建一个新的 URI，但使用 Route 对象的 URI 属性进行更新。新 URI 放置在 ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR 交换属性中。

如果 URI 具有方案前缀，例如 lb:ws://serviceid，则 lb 方案将从 URI 中剥离并放置在 ServerWebExchangeUtils.GATEWAY_SCHEME_PREFIX_ATTR 中，以便稍后在过滤器链中使用。

### 7.7. Websocket 路由过滤器

如果位于 ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR 交换属性中的 URL 具有 ws 或 wss 方案，则 websocket 路由过滤器运行。它使用 Spring WebSocket 基础结构向下游转发 websocket 请求。

您可以通过在 URI 前加上 lb 来对 websockets 进行负载平衡，例如 lb:ws://serviceid。

如果你使用 SockJS 作为普通 HTTP 的后备，你应该配置一个普通的 HTTP 路由以及 websocket 路由。

以下清单配置了一个 websocket 路由过滤器：

**Example 57. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      # SockJS route
      - id: websocket_sockjs_route
        uri: http://localhost:3001
        predicates:
        - Path=/websocket/info/**
      # Normal Websocket route
      - id: websocket_route
        uri: ws://localhost:3001
        predicates:
        - Path=/websocket/**
```

### 7.8.网关指标过滤器

要启用网关指标，请将 spring-boot-starter-actuator 添加为项目依赖项。然后，默认情况下，只要属性 spring.cloud.gateway.metrics.enabled 未设置为 false，网关指标过滤器就会运行。此过滤器添加了一个名为 gateway.requests 的计时器指标，并带有以下标签：

- `routeId`: 路由标识。
- `routeUri`: API 路由到的 URI。
- `outcome`: 结果，由 HttpStatus.Series 分类。
- `status`: 返回给客户端的请求的 HTTP 状态。
- `httpStatusCode`: 返回给客户端的请求的 HTTP 状态。
- `httpMethod`: 用于请求的 HTTP 方法。

然后可以从 /actuator/metrics/gateway.requests 抓取这些指标，并且可以轻松地与 Prometheus 集成以创建 Grafana 仪表板。

要启用 prometheus 端点，请将 micrometer-registry-prometheus 添加为项目依赖项。

### 7.9.将交换标记为已路由

网关路由 ServerWebExchange 后，它通过将 gatewayAlreadyRouted 添加到交换属性来将该交换标记为“已路由”。一旦请求被标记为路由，其他路由过滤器将不会再次路由该请求，实质上是跳过过滤器。有一些方便的方法可用于将交换标记为已路由或检查交换是否已被路由。

- ServerWebExchangeUtils.isAlreadyRouted 接受一个 ServerWebExchange 对象并检查它是否已被“路由”。
- ServerWebExchangeUtils.setAlreadyRouted 接受一个 ServerWebExchange 对象并将其标记为“已路由”。

## 8. HttpHeadersFilters

HttpHeadersFilters 在向下游发送请求之前应用于请求，例如在 NettyRoutingFilter 中。

### 8.1.Forwarded Headers过滤器

Forwarded Headers过滤器创建转发头以发送到下游服务。它将当前请求的 Host 标头、方案和端口添加到任何现有的 Forwarded 标头中。

### 8.2. RemoveHopByHop header过滤器

RemoveHopByHop 标头过滤器从转发的请求中删除标头。删除的默认标头列表来自 IETF。

默认删除的headers是：

- Connection
- Keep-Alive
- Proxy-Authenticate
- Proxy-Authorization
- TE
- Trailer
- Transfer-Encoding
- Upgrade

要更改此设置，请将 spring.cloud.gateway.filter.remove-hop-by-hop.headers 属性设置为要删除的标头名称列表。

### 8.3. XForwarded header过滤器

XForwarded 标头过滤器创建各种 X-Forwarded-* 标头以发送到下游服务。它使用当前请求的主机头、方案、端口和路径来创建各种头。

可以通过以下布尔属性（默认为 true）控制单个header的创建：

- `spring.cloud.gateway.x-forwarded.for-enabled`
- `spring.cloud.gateway.x-forwarded.host-enabled`
- `spring.cloud.gateway.x-forwarded.port-enabled`
- `spring.cloud.gateway.x-forwarded.proto-enabled`
- `spring.cloud.gateway.x-forwarded.prefix-enabled`

附加多个header可以由以下布尔属性控制（默认为 true）：

- `spring.cloud.gateway.x-forwarded.for-append`
- `spring.cloud.gateway.x-forwarded.host-append`
- `spring.cloud.gateway.x-forwarded.port-append`
- `spring.cloud.gateway.x-forwarded.proto-append`
- `spring.cloud.gateway.x-forwarded.prefix-append`

## 9. TLS 和 SSL

网关可以通过遵循通常的 Spring 服务器配置来侦听 HTTPS 上的请求。以下示例显示了如何执行此操作：

**Example 58. application.yml**

```yaml
server:
  ssl:
    enabled: true
    key-alias: scg
    key-store-password: scg1234
    key-store: classpath:scg-keystore.p12
    key-store-type: PKCS12
```

您可以将网关路由路由到 HTTP 和 HTTPS 后端。如果要路由到 HTTPS 后端，则可以使用以下配置将网关配置为信任所有下游证书：

**Example 59. application.yml**

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        ssl:
          useInsecureTrustManager: true
```

使用不安全的信任管理器不适合生产。对于生产部署，您可以使用一组可以信任的已知证书配置网关，并使用以下配置：

**Example 60. application.yml**

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        ssl:
          trustedX509Certificates:
          - cert1.pem
          - cert2.pem
```

如果 Spring Cloud Gateway 未提供受信任的证书，则使用默认信任存储（您可以通过设置 javax.net.ssl.trustStore 系统属性来覆盖）。

### 9.1. TLS 握手

网关维护一个客户端池，用于路由到后端。通过 HTTPS 通信时，客户端会发起 TLS 握手。许多超时与此握手相关联。您可以配置这些超时可以配置（默认显示）如下：

**Example 61. application.yml**

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        ssl:
          handshake-timeout-millis: 10000
          close-notify-flush-timeout-millis: 3000
          close-notify-read-timeout-millis: 0
```

## 10. 配置

Spring Cloud Gateway 的配置由一组 RouteDefinitionLocator 实例驱动。以下清单显示了 RouteDefinitionLocator 接口的定义：

**Example 62. RouteDefinitionLocator.java**

```java
public interface RouteDefinitionLocator {
    Flux<RouteDefinition> getRouteDefinitions();
}
```

默认情况下，PropertiesRouteDefinitionLocator 使用 Spring Boot 的 @ConfigurationProperties 机制加载属性。

较早的配置示例都使用使用位置参数而不是命名参数的快捷表示法。下面两个例子是等价的：

**Example 63. application.yml**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: setstatus_route
        uri: https://example.org
        filters:
        - name: SetStatus
          args:
            status: 401
      - id: setstatusshortcut_route
        uri: https://example.org
        filters:
        - SetStatus=401
```

对于网关的某些用途，属性就足够了，但某些生产用例受益于从外部源（例如数据库）加载配置。未来的里程碑版本将具有基于 Spring Data Repositories 的 RouteDefinitionLocator 实现，例如 Redis、MongoDB 和 Cassandra。

