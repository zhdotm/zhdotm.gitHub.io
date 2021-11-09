---
title: SpringCloud编程模型
date: 2021-11-09 08:12:43
tags: SpringCloud
categories: SpringCloud
---


# SpringCloud编程模型

## 云原生应用

Cloud Native 是一种应用程序开发风格，它鼓励在持续交付和价值驱动的开发领域轻松采用最佳实践。一个相关的学科是构建 12 要素应用程序，其中开发实践与交付和运营目标保持一致 —— 例如，通过使用声明式编程以及管理和监控。 Spring Cloud 以多种特定方式促进了这些开发风格。起点是一组特性，分布式系统中的所有组件都需要轻松访问这些特性。

其中许多功能都包含在 Spring Boot 中，Spring Cloud 在其上构建。 Spring Cloud 提供了更多功能作为两个库：Spring Cloud Context 和 Spring Cloud Commons。 Spring Cloud Context 为 Spring Cloud 应用程序的 ApplicationContext 提供实用程序和特殊服务（引导上下文、加密、刷新范围和环境端点）。 Spring Cloud Commons 是在不同的 Spring Cloud 实现（例如 Spring Cloud Netflix 和 Spring Cloud Consul）中使用的一组抽象和通用类。

如果由于“非法密钥大小”而出现异常并且您使用 Sun 的 JDK，则需要安装 Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files。有关更多信息，请参阅以下链接：

- [Java 6 JCE](https://www.oracle.com/technetwork/java/javase/downloads/jce-6-download-429243.html)
- [Java 7 JCE](https://www.oracle.com/technetwork/java/javase/downloads/jce-7-download-432124.html)
- [Java 8 JCE](https://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html)

对于您使用的任何 JRE/JDK x64/x86 版本，将文件解压缩到 JDK/jre/lib/security 文件夹中。

### 1. Spring Cloud Context：应用程序上下文服务

Spring Boot 对如何使用 Spring 构建应用程序有自己的看法。例如，它具有常见配置文件的常规位置，并具有用于常见管理和监控任务的端点。 Spring Cloud 在此基础上构建，并添加了一些系统中许多组件会使用或偶尔需要的功能。

#### 1.1. Bootstrap 应用程序上下文

Spring Cloud 应用程序通过创建“引导程序”上下文来运行，该上下文是主应用程序的父上下文。此上下文负责从外部源加载配置属性并解密本地外部配置文件中的属性。这两个上下文共享一个环境，它是任何 Spring 应用程序的外部属性的来源。默认情况下，引导属性（不是 bootstrap.properties，而是在引导阶段加载的属性）以高优先级添加，因此它们不能被本地配置覆盖。

引导上下文使用与主应用程序上下文不同的约定来定位外部配置。您可以使用 bootstrap.yml 代替 application.yml（或 .properties），将引导程序和主上下文的外部配置很好地分开。以下清单显示了一个示例：

示例 1. bootstrap.yml

```yaml
spring:
  application:
    name: foo
  cloud:
    config:
      uri: ${SPRING_CONFIG_URI:http://localhost:8888}
```

如果您的应用程序需要来自服务器的任何特定于应用程序的配置，最好设置 spring.application.name（在 bootstrap.yml 或 application.yml 中）。要将属性 spring.application.name 用作应用程序的上下文 ID，您必须在 bootstrap.[properties|yml]中设置它。 。

如果要检索特定的配置文件配置，还应该在 bootstrap.[properties|yml] 中设置 spring.profiles.active。

您可以通过设置 spring.cloud.bootstrap.enabled=false （例如，在系统属性中）来完全禁用引导过程。

#### 1.2.应用程序上下文层次结构

如果您从 SpringApplication 或 SpringApplicationBuilder 构建应用程序上下文，则 Bootstrap 上下文将作为父级添加到该上下文。 Spring 的一个特性是子上下文从其父上下文继承属性源和配置文件，因此“主”应用程序上下文包含额外的属性源，与在没有 Spring Cloud Config 的情况下构建相同的上下文相比。额外的财产来源是：

- “bootstrap”：如果在 bootstrap 上下文中找到任何 PropertySourceLocators 并且它们具有非空属性，则会出现一个具有高优先级的可选 CompositePropertySource。一个例子是来自 Spring Cloud Config Server 的属性。有关如何自定义此属性源的内容，请参阅“自定义 Bootstrap 属性源”。
- “applicationConfig: [classpath:bootstrap.yml]”（以及相关文件，如果 Spring 配置文件处于活动状态）：如果您有 bootstrap.yml（或 .properties），这些属性用于配置引导程序上下文。然后当它的父级被设置时，它们被添加到子上下文中。它们的优先级低于 application.yml（或 .properties）以及作为创建 Spring Boot 应用程序过程的正常部分添加到子项的任何其他属性源。有关如何自定义这些属性源的内容，请参阅“更改 Bootstrap 属性的位置”。

由于属性源的排序规则，“引导”条目优先。但是，请注意这些不包含来自 bootstrap.yml 的任何数据，它具有非常低的优先级但可用于设置默认值。

您可以通过设置您创建的任何 ApplicationContext 的父上下文来扩展上下文层次结构 — 例如，通过使用它自己的接口或使用 SpringApplicationBuilder 便捷方法（parent()、child() 和sibling()）。引导上下文是您自己创建的最高级祖先的父级。层次结构中的每个上下文都有自己的“引导程序”（可能是空的）属性源，以避免无意中将值从父级提升到其后代。如果有配置服务器，层次结构中的每个上下文也可以（原则上）具有不同的 spring.application.name，因此，具有不同的远程属性源。普通 Spring 应用程序上下文行为规则适用于属性解析：来自子上下文的属性覆盖父上下文中的属性，按名称以及属性源名称。 （如果子级具有与父级同名的属性源，则父级的值不包含在子级中）。

请注意， SpringApplicationBuilder 允许您在整个层次结构中共享环境，但这不是默认设置。因此，兄弟上下文（特别是）不需要具有相同的配置文件或属性源，即使它们可能与其父级共享共同的值。

#### 1.3.更改 Bootstrap 属性的位置

可以通过设置 spring.cloud.bootstrap.name（默认：bootstrap）、spring.cloud.bootstrap.location（默认：空）或 spring.cloud.bootstrap.additional-location 来指定 bootstrap.yml（或 .properties）位置（默认：空） — 例如，在系统属性中。

这些属性的行为类似于具有相同名称的 spring.config.* 变体。使用 spring.cloud.bootstrap.location 替换默认位置，仅使用指定的位置。要将位置添加到默认列表中，可以使用 spring.cloud.bootstrap.additional-location。事实上，它们用于通过在 Environment 中设置这些属性来设置 bootstrap ApplicationContext。如果有一个活动配置文件（来自 spring.profiles.active 或通过您正在构建的上下文中的环境 API），该配置文件中的属性也会被加载，就像在常规 Spring Boot 应用程序中一样 — 例如，来自 bootstrap -development.properties 用于开发配置文件。

#### 1.4.覆盖远程属性的值

由引导上下文添加到应用程序的属性源通常是“远程的”（例如，来自 Spring Cloud Config Server）。默认情况下，它们不能在本地被覆盖。如果你想让你的应用程序用他们自己的系统属性或配置文件覆盖远程属性，远程属性源必须通过设置 spring.cloud.config.allowOverride=true 来授予它权限（在本地设置它不起作用） .一旦设置了该标志，两个更细粒度的设置将控制与系统属性和应用程序本地配置相关的远程属性的位置：

- spring.cloud.config.overrideNone=true：从任何本地属性源覆盖。
- spring.cloud.config.overrideSystemProperties=false：只有系统属性、命令行参数和环境变量（而不是本地配置文件）应该覆盖远程设置。

#### 1.5.自定义引导配置

通过在名为 org.springframework.cloud.bootstrap.BootstrapConfiguration 的键下向 /META-INF/spring.factories 添加条目，可以将引导上下文设置为执行您喜欢的任何操作。这包含用于创建上下文的 Spring @Configuration 类的逗号分隔列表。您希望在主应用程序上下文中可用于自动装配的任何 bean 都可以在此处创建。 ApplicationContextInitializer 类型的@Beans 有一个特殊的契约。如果要控制启动顺序，可以用@Order 注解标记类（默认顺序是最后）。

添加自定义 BootstrapConfiguration 时，请注意您添加的类不会错误地@ComponentScanned 到您的“主”应用程序上下文中，在那里可能不需要它们。为引导配置类使用单独的包名称，并确保该名称尚未被 @ComponentScan 或 @SpringBootApplication 注释的配置类覆盖。

bootstrap 过程通过将初始化器注入主 SpringApplication 实例（这是正常的 Spring Boot 启动序列，无论它作为独立应用程序运行还是部署在应用程序服务器中）结束。首先，从 spring.factories 中的类创建引导上下文。然后，ApplicationContextInitializer 类型的所有@Beans 在启动之前被添加到主 SpringApplication 中。

#### 1.6.自定义 Bootstrap 属性源

bootstrap 进程添加的外部配置的默认属性源是 Spring Cloud Config Server，但您可以通过将 PropertySourceLocator 类型的 bean 添加到 bootstrap 上下文（通过 spring.factories）来添加其他源。例如，您可以插入来自不同服务器或数据库的其他属性。

例如，请考虑以下自定义定位器：

```java
@Configuration
public class CustomPropertySourceLocator implements PropertySourceLocator {

    @Override
    public PropertySource<?> locate(Environment environment) {
        return new MapPropertySource("customProperty",
                Collections.<String, Object>singletonMap("property.from.sample.custom.source", "worked as intended"));
    }

}
```

传入的 Environment 是即将创建的 ApplicationContext 的环境 — 换句话说，我们为其提供附加属性源的环境。它已经拥有普通的 Spring Boot 提供的属性源，因此您可以使用它们来定位特定于此环境的属性源（例如，通过在 spring.application.name 上键入它，就像在默认的 Spring Cloud Config Server 中所做的那样属性源定位器）。

如果您在其中创建一个包含此类的 jar，然后添加包含以下设置的 META-INF/spring.factories，则 customProperty PropertySource 将出现在其类路径中包含该 jar 的任何应用程序中：

```java
org.springframework.cloud.bootstrap.BootstrapConfiguration=sample.custom.CustomPropertySourceLocator
```

#### 1.7.日志配置

如果你使用 Spring Boot 来配置日志设置，你应该把这个配置放在 bootstrap.[yml|properties] 如果您希望它适用于所有事件。

为了让 Spring Cloud 正确初始化日志配置，您不能使用自定义前缀。例如，在初始化日志系统时，Spring Cloud 无法识别使用 custom.loggin.logpath。

#### 1.8.环境变化

应用程序侦听 EnvironmentChangeEvent 并以几种标准方式对更改做出反应（额外的 ApplicationListeners 可以以正常方式添加为 @Beans）。当观察到 EnvironmentChangeEvent 时，它具有已更改的键值列表，应用程序使用这些值：

- 在上下文中重新绑定任何 @ConfigurationProperties bean。
- 为 logging.level.* 中的任何属性设置记录器级别。

请注意，默认情况下，Spring Cloud Config Client 不会轮询环境中的更改。通常，我们不建议使用这种方法来检测更改（尽管您可以使用 @Scheduled 注释进行设置）。如果您有横向扩展的客户端应用程序，最好将 EnvironmentChangeEvent 广播到所有实例，而不是让它们轮询更改（例如，通过使用 Spring Cloud Bus）。

EnvironmentChangeEvent 涵盖了一大类刷新用例，只要您可以实际更改 Environment 并发布事件即可。请注意，这些 API 是公共的并且是核心 Spring 的一部分）。您可以通过访问 /configprops 端点（标准 Spring Boot Actuator 功能）来验证更改是否绑定到 @ConfigurationProperties bean。例如，DataSource 可以在运行时更改其 maxPoolSize（Spring Boot 创建的默认 DataSource 是一个 @ConfigurationProperties bean）并动态增加容量。重新绑定 @ConfigurationProperties 不涵盖另一大类用例，在这些用例中，您需要对刷新进行更多控制，并且需要对整个 ApplicationContext 进行原子性更改。为了解决这些问题，我们有@RefreshScope。

#### 1.9.刷新范围

当配置发生变化时，标记为@RefreshScope 的 Spring @Bean 会得到特殊处理。此功能解决了有状态 bean 的问题，这些 bean 仅在初始化时注入其配置。例如，如果在通过环境更改数据库 URL 时 DataSource 具有打开的连接，您可能希望这些连接的持有者能够完成他们正在做的事情。然后，下一次从池中借用连接时，它会获得一个带有新 URL 的连接。

有时，甚至可能必须在某些只能初始化一次的 bean 上应用 @RefreshScope 注释。如果 bean 是“不可变的”，则必须使用 @RefreshScope 注释 bean 或在属性键下指定类名：spring.cloud.refresh.extra-refreshable。

如果你有一个作为 HikariDataSource 的 DataSource bean，它不能被刷新。这是 spring.cloud.refresh.never-refreshable 的默认值。如果需要刷新，请选择不同的 DataSource 实现。

刷新作用域 bean 是在使用时（即调用方法时）进行初始化的惰性代理，作用域充当已初始化值的缓存。要强制 bean 在下一次方法调用时重新初始化，您必须使其缓存条目无效。

RefreshScope 是上下文中的一个 bean，并且有一个公共 refreshAll() 方法通过清除目标缓存来刷新范围内的所有 bean。 /refresh 端点公开了此功能（通过 HTTP 或 JMX）。要按名称刷新单个 bean，还有一个 refresh(String) 方法。

要公开 /refresh 端点，您需要向应用程序添加以下配置：

```yaml
management:
  endpoints:
    web:
      exposure:
        include: refresh
```

@RefreshScope 在 @Configuration 类上工作（技术上），但它可能会导致令人惊讶的行为。例如，这并不意味着该类中定义的所有@Beans 本身都在@RefreshScope 中。具体来说，任何依赖于这些 bean 的东西都不能依赖于在启动刷新时更新它们，除非它本身在 @RefreshScope 中。在这种情况下，它会在刷新时重建，并重新注入其依赖项。那时，它们从刷新的@Configuration 重新初始化）。

#### 1.10.加密和解密

Spring Cloud 有一个 Environment 预处理器，用于在本地解密属性值。它遵循与 Spring Cloud Config Server 相同的规则，并通过 encrypt.* 具有相同的外部配置。因此，您可以使用 {cipher}* 形式的加密值，并且只要存在有效密钥，它们就会在主应用程序上下文获得环境设置之前被解密。要在应用程序中使用加密功能，您需要在类路径中包含 Spring Security RSA（Maven 坐标：org.springframework.security:spring-security-rsa），并且您还需要在 JVM 中使用完整的 JCE 扩展.

如果由于“非法密钥大小”而出现异常并且您使用 Sun 的 JDK，则需要安装 Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files。有关更多信息，请参阅以下链接：

- [Java 6 JCE](https://www.oracle.com/technetwork/java/javase/downloads/jce-6-download-429243.html)
- [Java 7 JCE](https://www.oracle.com/technetwork/java/javase/downloads/jce-7-download-432124.html)
- [Java 8 JCE](https://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html)

对于您使用的任何 JRE/JDK x64/x86 版本，将文件解压缩到 JDK/jre/lib/security 文件夹中。

#### 1.11.端点

对于 Spring Boot Actuator 应用程序，可以使用一些额外的管理端点。您可以使用：

- POST 到 /actuator/env 以更新环境并重新绑定 @ConfigurationProperties 和日志级别。要启用此端点，您必须设置 management.endpoint.env.post.enabled=true。
- /actuator/refresh 重新加载引导上下文并刷新@RefreshScope bean。
- /actuator/restart 关闭 ApplicationContext 并重新启动它（默认情况下禁用）。
- /actuator/pause 和 /actuator/resume 用于调用生命周期方法（ApplicationContext 上的 stop() 和 start()）。

如果您禁用 /actuator/restart 端点，那么 /actuator/pause 和 /actuator/resume 端点也将被禁用，因为它们只是 /actuator/restart 的一个特例。

### 2. Spring Cloud Commons：通用抽象

服务发现、负载平衡和断路器等模式适合一个公共抽象层，所有 Spring Cloud 客户端都可以使用该抽象层，独立于实现（例如，使用 Eureka 或 Consul 进行发现）。

#### 2.1. @EnableDiscoveryClient 注解

Spring Cloud Commons 提供了 @EnableDiscoveryClient 注解。这会寻找带有 META-INF/spring.factories 的 DiscoveryClient 和 ReactiveDiscoveryClient 接口的实现。发现客户端的实现在 org.springframework.cloud.client.discovery.EnableDiscoveryClient 键下的 spring.factories 中添加了一个配置类。 DiscoveryClient 实现的示例包括 Spring Cloud Netflix Eureka、Spring Cloud Consul Discovery 和 Spring Cloud Zookeeper Discovery。

默认情况下，Spring Cloud 将提供阻塞和反应式服务发现客户端。您可以通过设置 spring.cloud.discovery.blocking.enabled=false 或 spring.cloud.discovery.reactive.enabled=false 轻松禁用阻塞和/或反应客户端。要完全禁用服务发现，您只需设置 spring.cloud.discovery.enabled=false。

默认情况下， DiscoveryClient 的实现会自动向远程发现服务器注册本地 Spring Boot 服务器。可以通过在 @EnableDiscoveryClient 中设置 autoRegister=false 来禁用此行为。

不再需要@EnableDiscoveryClient。您可以在类路径上放置一个 DiscoveryClient 实现，以使 Spring Boot 应用程序向服务发现服务器注册。

##### 2.1.1.健康指标

Commons 自动配置以下 Spring Boot 健康指标。

###### 发现客户端健康指标[DiscoveryClientHealthIndicator](https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/#discoveryclienthealthindicator)

此运行状况指标基于当前注册的 DiscoveryClient 实现。

- 要完全禁用，请设置 spring.cloud.discovery.client.health-indicator.enabled=false。
- 要禁用描述字段，请设置 spring.cloud.discovery.client.health-indicator.include-description=false。否则，它可能会冒泡作为汇总的 HealthIndicator 的描述。
- 要禁用服务检索，请设置 spring.cloud.discovery.client.health-indicator.use-services-query=false。默认情况下，指标调用客户端的 getServices 方法。在具有许多注册服务的部署中，在每次检查期间检索所有服务的成本可能太高。这将跳过服务检索，而是使用客户端的探测方法。

###### [DiscoveryCompositeHealthContributor](https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/#discoverycompositehealthcontributor)

此复合健康指标基于所有已注册的 DiscoveryHealthIndicator bean。要禁用，请设置 spring.cloud.discovery.client.composite-indicator.enabled=false。

##### 2.1.2.Ordering `DiscoveryClient` 实例

DiscoveryClient 接口扩展了 Ordered。这在使用多个发现客户端时很有用，因为它允许您定义返回的发现客户端的顺序，类似于如何对 Spring 应用程序加载的 bean 进行排序。默认情况下，任何 DiscoveryClient 的顺序设置为 0。如果您想为自定义 DiscoveryClient 实现设置不同的顺序，您只需覆盖 getOrder() 方法，以便它返回适合您设置的值。除此之外，您可以使用属性来设置 Spring Cloud 提供的 DiscoveryClient 实现的顺序，其中包括 ConsulDiscoveryClient、EurekaDiscoveryClient 和 ZookeeperDiscoveryClient。为此，您只需将 spring.cloud.{clientIdentifier}.discovery.order （或 Eureka 的 eureka.client.order）属性设置为所需的值。

##### [2.1.3. SimpleDiscoveryClient](https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/#simplediscoveryclient)

如果类路径中没有 Service-Registry-backed DiscoveryClient，将使用 SimpleDiscoveryClient 实例，它使用属性来获取有关服务和实例的信息。

有关可用实例的信息应通过以下格式的属性传递给： spring.cloud.discovery.client.simple.instances.service1[0].uri=http://s11:8080，其中 spring.cloud.discovery .client.simple.instances 是公共前缀，然后service1代表所涉及的服务的ID，而[0]代表实例的索引号（如示例中可见，索引从0开始），然后是uri 的值是实例可用的实际 URI。

#### [2.2. ServiceRegistry](https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/#serviceregistry)

Commons 现在提供了一个 ServiceRegistry 接口，该接口提供 register(Registration) 和 deregister(Registration) 等方法，让您可以提供自定义注册服务。注册是一个标记界面。

以下示例显示了正在使用的 ServiceRegistry：

```java
@Configuration
@EnableDiscoveryClient(autoRegister=false)
public class MyConfiguration {
    private ServiceRegistry registry;

    public MyConfiguration(ServiceRegistry registry) {
        this.registry = registry;
    }

    // called through some external process, such as an event or a custom actuator endpoint
    public void register() {
        Registration registration = constructRegistration();
        this.registry.register(registration);
    }
}
```

每个 ServiceRegistry 实现都有自己的 Registry 实现。

- ZookeeperRegistration 与 ZookeeperServiceRegistry 一起使用 
- EurekaRegistration 与 EurekaServiceRegistry 一起使用 
- ConsulRegistration 与 ConsulServiceRegistry 一起使用

如果您使用 ServiceRegistry 接口，您将需要为您正在使用的 ServiceRegistry 实现传递正确的 Registry 实现。

##### [2.2.1. ServiceRegistry Auto-Registration](https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/#serviceregistry-auto-registration)

默认情况下，ServiceRegistry 实现会自动注册正在运行的服务。要禁用该行为，您可以设置： * @EnableDiscoveryClient(autoRegister=false) 以永久禁用自动注册。 * spring.cloud.service-registry.auto-registration.enabled=false 通过配置禁用行为。

###### [ServiceRegistry Auto-Registration Events](https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/#serviceregistry-auto-registration-events)

服务自动注册时将触发两个事件。第一个事件称为 InstancePreRegisteredEvent，在注册服务之前触发。第二个事件称为 InstanceRegisteredEvent，在注册服务后触发。您可以注册一个 ApplicationListener(s) 来监听和响应这些事件。

如果 spring.cloud.service-registry.auto-registration.enabled 属性设置为 false，则不会触发这些事件。

##### [ 2.2.2. Service Registry Actuator Endpoint](https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/#service-registry-actuator-endpoint)

Spring Cloud Commons 提供了一个 /service-registry 执行器端点。此端点依赖于 Spring 应用程序上下文中的注册 bean。使用 GET 调用 /service-registry 会返回注册的状态。对具有 JSON 正文的同一端点使用 POST 会将当前注册的状态更改为新值。 JSON 正文必须包含具有首选值的状态字段。请参阅更新状态和状态返回值时用于允许值的 ServiceRegistry 实现的文档。例如，Eureka 支持的状态是 UP、DOWN、OUT_OF_SERVICE 和 UNKNOWN。

#### 2.3. Spring RestTemplate 作为负载均衡器客户端

您可以配置 RestTemplate 以使用负载平衡器客户端。要创建负载平衡的 RestTemplate，请创建 RestTemplate @Bean 并使用 @LoadBalanced 限定符，如以下示例所示：

```java
@Configuration
public class MyConfiguration {

    @LoadBalanced
    @Bean
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

public class MyClass {
    @Autowired
    private RestTemplate restTemplate;

    public String doOtherStuff() {
        String results = restTemplate.getForObject("http://stores/stores", String.class);
        return results;
    }
}
```

RestTemplate bean 不再通过自动配置创建。个人应用程序必须创建它。

URI 需要使用虚拟主机名（即服务名，而不是主机名）。 BlockingLoadBalancerClient 用于创建完整的物理地址。

要使用负载平衡的 RestTemplate，您需要在类路径中有一个负载平衡器实现。将 Spring Cloud LoadBalancer starter 添加到您的项目中以便使用它。

#### 2.4. Spring WebClient 作为负载均衡器客户端

您可以将 WebClient 配置为自动使用负载平衡器客户端。要创建负载均衡的 WebClient，请创建一个 WebClient.Builder @Bean 并使用 @LoadBalanced 限定符，如下所示：

```java
@Configuration
public class MyConfiguration {

    @Bean
    @LoadBalanced
    public WebClient.Builder loadBalancedWebClientBuilder() {
        return WebClient.builder();
    }
}

public class MyClass {
    @Autowired
    private WebClient.Builder webClientBuilder;

    public Mono<String> doOtherStuff() {
        return webClientBuilder.build().get().uri("http://stores/stores")
                        .retrieve().bodyToMono(String.class);
    }
}
```

URI 需要使用虚拟主机名（即服务名，而不是主机名）。 Spring Cloud LoadBalancer 用于创建完整的物理地址。

如果你想使用@LoadBalanced WebClient.Builder，你需要在类路径中有一个负载均衡器实现。我们建议您将 Spring Cloud LoadBalancer starter 添加到您的项目中。然后，在下面使用 ReactiveLoadBalancer。

##### 2.4.1.重试失败的请求

负载平衡的 RestTemplate 可以配置为重试失败的请求。默认情况下，此逻辑被禁用。对于非响应式版本（使用 RestTemplate），您可以通过将 Spring Retry 添加到应用程序的类路径来启用它。对于响应式版本（使用 WebTestClient），您需要设置 `spring.cloud.loadbalancer.retry.enabled=true。

如果您想在类路径上使用 Spring Retry 或 Reactive Retry 禁用重试逻辑，您可以设置 spring.cloud.loadbalancer.retry.enabled=false。

对于非反应式实现，如果您想在重试中实现 BackOffPolicy，则需要创建一个 LoadBalancedRetryFactory 类型的 bean 并覆盖 createBackOffPolicy() 方法。

对于反应式实现，您只需要通过将 spring.cloud.loadbalancer.retry.backoff.enabled 设置为 false 来启用它。

您可以设置：

- spring.cloud.loadbalancer.retry.maxRetriesOnSameServiceInstance - 指示应在同一个 ServiceInstance 上重试请求的次数（为每个选定的实例单独计数） 
- spring.cloud.loadbalancer.retry.maxRetriesOnNextServiceInstance - 指示应重试新选择的 ServiceInstance 请求的次数 
- spring.cloud.loadbalancer.retry.retryableStatusCodes - 始终重试失败请求的状态代码。

对于反应式实现，您还可以设置： - spring.cloud.loadbalancer.retry.backoff.minBackoff - 设置最小退避持续时间（默认为 5 毫秒） - spring.cloud.loadbalancer.retry.backoff.maxBackoff - 设置最大退避持续时间（默认情况下，最大长值毫秒） - spring.cloud.loadbalancer.retry.backoff.jitter - 设置用于计算每次调用的实际退避持续时间的抖动（默认情况下，0.5）。

对于反应式实现，您还可以实现自己的 LoadBalancerRetryPolicy 以更详细地控制负载平衡的调用重试。

对于负载平衡重试，默认情况下，我们使用 RetryAwareServiceInstanceListSupplier 包装 ServiceInstanceListSupplier bean，以从先前选择的实例中选择一个不同的实例（如果可用）。您可以通过将 spring.cloud.loadbalancer.retry.avoidPreviousInstance 的值设置为 false 来禁用此行为。

```java
@Configuration
public class MyConfiguration {
    @Bean
    LoadBalancedRetryFactory retryFactory() {
        return new LoadBalancedRetryFactory() {
            @Override
            public BackOffPolicy createBackOffPolicy(String service) {
                return new ExponentialBackOffPolicy();
            }
        };
    }
}
```

如果您想将一个或多个 RetryListener 实现添加到您的重试功能中，您需要创建一个 LoadBalancedRetryListenerFactory 类型的 bean 并返回您想用于给定服务的 RetryListener 数组，如以下示例所示：

```java
@Configuration
public class MyConfiguration {
    @Bean
    LoadBalancedRetryListenerFactory retryListenerFactory() {
        return new LoadBalancedRetryListenerFactory() {
            @Override
            public RetryListener[] createRetryListeners(String service) {
                return new RetryListener[]{new RetryListener() {
                    @Override
                    public <T, E extends Throwable> boolean open(RetryContext context, RetryCallback<T, E> callback) {
                        //TODO Do you business...
                        return true;
                    }

                    @Override
                     public <T, E extends Throwable> void close(RetryContext context, RetryCallback<T, E> callback, Throwable throwable) {
                        //TODO Do you business...
                    }

                    @Override
                    public <T, E extends Throwable> void onError(RetryContext context, RetryCallback<T, E> callback, Throwable throwable) {
                        //TODO Do you business...
                    }
                }};
            }
        };
    }
}
```

#### 2.5.多个 RestTemplate 对象

如果您想要一个非负载均衡的 RestTemplate，请创建一个 RestTemplate bean 并注入它。要访问负载平衡的 RestTemplate，请在创建 @Bean 时使用 @LoadBalanced 限定符，如以下示例所示：

```java
@Configuration
public class MyConfiguration {

    @LoadBalanced
    @Bean
    WebClient.Builder loadBalanced() {
        return WebClient.builder();
    }

    @Primary
    @Bean
    WebClient.Builder webClient() {
        return WebClient.builder();
    }
}

public class MyClass {
    @Autowired
    private WebClient.Builder webClientBuilder;

    @Autowired
    @LoadBalanced
    private WebClient.Builder loadBalanced;

    public Mono<String> doOtherStuff() {
        return loadBalanced.build().get().uri("http://stores/stores")
                        .retrieve().bodyToMono(String.class);
    }

    public Mono<String> doStuff() {
        return webClientBuilder.build().get().uri("http://example.com")
                        .retrieve().bodyToMono(String.class);
    }
}
```

#### 2.7. Spring WebFlux WebClient 作为负载均衡器客户端

Spring WebFlux 可以使用反应式和非反应式 WebClient 配置，如主题所述：

- 带有 ReactorLoadBalancerExchangeFilterFunction 的 Spring WebFlux WebClient
- [负载平衡器交换过滤器功能负载平衡器交换过滤器功能]

##### 2.7.1.带有 ReactorLoadBalancerExchangeFilterFunction 的 Spring WebFlux WebClient

您可以将 WebClient 配置为使用 ReactiveLoadBalancer。如果您将 Spring Cloud LoadBalancer starter 添加到您的项目中并且如果 spring-webflux 在类路径上，则 ReactorLoadBalancerExchangeFilterFunction 是自动配置的。以下示例显示如何配置 WebClient 以使用反应式负载均衡器：

```java
public class MyClass {
    @Autowired
    private ReactorLoadBalancerExchangeFilterFunction lbFunction;

    public Mono<String> doOtherStuff() {
        return WebClient.builder().baseUrl("http://stores")
            .filter(lbFunction)
            .build()
            .get()
            .uri("/stores")
            .retrieve()
            .bodyToMono(String.class);
    }
}
```

URI 需要使用虚拟主机名（即服务名，而不是主机名）。 ReactorLoadBalancer 用于创建完整的物理地址。

##### 2.7.2.带有非反应式负载均衡器客户端的 Spring WebFlux WebClient

如果 spring-webflux 在类路径上，LoadBalancerExchangeFilterFunction 是自动配置的。但是请注意，这在后台使用了一个非反应式客户端。以下示例显示如何配置 WebClient 以使用负载均衡器：

```java
public class MyClass {
    @Autowired
    private LoadBalancerExchangeFilterFunction lbFunction;

    public Mono<String> doOtherStuff() {
        return WebClient.builder().baseUrl("http://stores")
            .filter(lbFunction)
            .build()
            .get()
            .uri("/stores")
            .retrieve()
            .bodyToMono(String.class);
    }
}
```

URI 需要使用虚拟主机名（即服务名，而不是主机名）。 LoadBalancerClient 用于创建完整的物理地址。

警告：此方法现已弃用。我们建议您使用带有反应式负载均衡器的 WebFlux。

#### 2.8.忽略网络接口

有时，忽略某些命名的网络接口很有用，以便它们可以从服务发现注册中排除（例如，在 Docker 容器中运行时）。可以设置正则表达式列表以导致所需的网络接口被忽略。以下配置忽略了 docker0 接口和所有以 veth 开头的接口：

示例 2. application.yml

```yaml
spring:
  cloud:
    inetutils:
      ignoredInterfaces:
        - docker0
        - veth.*
```

您还可以通过使用正则表达式列表强制仅使用指定的网络地址，如以下示例所示：

示例 3. bootstrap.yml

```yaml
spring:
  cloud:
    inetutils:
      preferredNetworks:
        - 192.168
        - 10.0
```

您还可以强制仅使用站点本地地址，如以下示例所示：

示例 4. application.yml

```yaml
spring:
  cloud:
    inetutils:
      useOnlySiteLocalInterfaces: true
```

有关什么构成站点本地地址的更多详细信息，请参阅 Inet4Address.html.isSiteLocalAddress()。

#### 2.9. HTTP 客户端工厂

Spring Cloud Commons 提供了用于创建 Apache HTTP 客户端 (ApacheHttpClientFactory) 和 OK HTTP 客户端 (OkHttpClientFactory) 的 bean。只有当 OK HTTP jar 位于类路径上时，才会创建 OkHttpClientFactory bean。此外，Spring Cloud Commons 提供了用于创建两个客户端使用的连接管理器的 bean：ApacheHttpClientConnectionManagerFactory 用于 Apache HTTP 客户端，OkHttpClientConnectionPoolFactory 用于 OK HTTP 客户端。如果您想自定义如何在下游项目中创建 HTTP 客户端，您可以提供您自己的这些 bean 的实现。此外，如果您提供类型为 HttpClientBuilder 或 OkHttpClient.Builder 的 bean，则默认工厂使用这些构建器作为返回到下游项目的构建器的基础。您还可以通过将 spring.cloud.httpclientfactories.apache.enabled 或 spring.cloud.httpclientfactories.ok.enabled 设置为 false 来禁用这些 bean 的创建。

#### 2.10.启用的功能

Spring Cloud Commons 提供了一个 /features 执行器端点。此端点返回类路径上可用的功能以及它们是否已启用。返回的信息包括功能类型、名称、版本和供应商。

##### 2.10.1.特征类型

有两种类型的“特征”：抽象的和命名的。

抽象特性是定义接口或抽象类以及创建实现的特性，例如 DiscoveryClient、LoadBalancerClient 或 LockService。抽象类或接口用于在上下文中查找该类型的 bean。显示的版本是 bean.getClass().getPackage().getImplementationVersion()。

命名特性是没有它们实现的特定类的特性。这些功能包括“断路器”、“API 网关”、“Spring Cloud Bus”等。这些功能需要一个名称和一个 bean 类型。

##### 2.10.2.声明功能

任何模块都可以声明任意数量的 HasFeature bean，如以下示例所示：

```java
@Bean
public HasFeatures commonsFeatures() {
  return HasFeatures.abstractFeatures(DiscoveryClient.class, LoadBalancerClient.class);
}

@Bean
public HasFeatures consulFeatures() {
  return HasFeatures.namedFeatures(
    new NamedFeature("Spring Cloud Bus", ConsulBusAutoConfiguration.class),
    new NamedFeature("Circuit Breaker", HystrixCommandAspect.class));
}

@Bean
HasFeatures localFeatures() {
  return HasFeatures.builder()
      .abstractFeature(Something.class)
      .namedFeature(new NamedFeature("Some Other Feature", Someother.class))
      .abstractFeature(Somethingelse.class)
      .build();
}
```

这些 bean 中的每一个都应该放在一个适当保护的 @Configuration 中。

#### 2.11. Spring Cloud 兼容性验证

由于部分用户在设置 Spring Cloud 应用程序时遇到问题，我们决定添加兼容性验证机制。如果您当前的设置与 Spring Cloud 要求不兼容，它将中断，并附上一份报告，显示究竟出了什么问题。

目前我们验证将哪个版本的 Spring Boot 添加到您的类路径中。

报告示例

```java
***************************
APPLICATION FAILED TO START
***************************

Description:

Your project setup is incompatible with our requirements due to following reasons:

- Spring Boot [2.1.0.RELEASE] is not compatible with this Spring Cloud release train


Action:

Consider applying the following actions:

- Change Spring Boot version to one of the following versions [1.2.x, 1.3.x] .
You can find the latest Spring Boot versions here [https://spring.io/projects/spring-boot#learn].
If you want to learn more about the Spring Cloud Release train compatibility, you can visit this page [https://spring.io/projects/spring-cloud#overview] and check the [Release Trains] section.
```

要禁用此功能，请将 spring.cloud.compatibility-verifier.enabled 设置为 false。如果要覆盖兼容的 Spring Boot 版本，只需使用逗号分隔的兼容 Spring Boot 版本列表设置 spring.cloud.compatibility-verifier.compatible-boot-versions 属性。

### 3. Spring Cloud 负载均衡器

Spring Cloud 提供了自己的客户端负载均衡器抽象和实现。对于负载均衡机制，添加了 ReactiveLoadBalancer 接口，并为其提供了基于 Round-Robin 和 Random 的实现。为了让实例从反应式 ServiceInstanceListSupplier 中选择。目前，我们支持 ServiceInstanceListSupplier 的基于服务发现的实现，该实现使用类路径中可用的发现客户端从服务发现中检索可用实例。

可以通过将 spring.cloud.loadbalancer.enabled 的值设置为 false 来禁用 Spring Cloud LoadBalancer。

#### 3.1.在负载平衡算法之间切换

默认使用的 ReactiveLoadBalancer 实现是 RoundRobinLoadBalancer。要为选定的服务或所有服务切换到不同的实现，您可以使用自定义 LoadBalancer 配置机制。

例如，可以通过@LoadBalancerClient 注解传递以下配置以切换到使用 RandomLoadBalancer：

```java
public class CustomLoadBalancerConfiguration {

    @Bean
    ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(Environment environment,
            LoadBalancerClientFactory loadBalancerClientFactory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(loadBalancerClientFactory
                .getLazyProvider(name, ServiceInstanceListSupplier.class),
                name);
    }
}
```

您作为 @LoadBalancerClient 或 @LoadBalancerClients 配置参数传递的类不应使用 @Configuration 进行注释或不在组件扫描范围内。

#### 3.2. Spring Cloud LoadBalancer 集成

为了方便使用 Spring Cloud LoadBalancer，我们提供了可与 WebClient 一起使用的 ReactorLoadBalancerExchangeFilterFunction 和与 RestTemplate 一起使用的 BlockingLoadBalancerClient。您可以在以下部分中查看更多信息和用法示例：

- Spring RestTemplate 作为负载均衡器客户端 
- Spring WebClient 作为负载均衡器客户端 
- 带有 ReactorLoadBalancerExchangeFilterFunction 的 Spring WebFlux WebClient

#### 3.3. Spring Cloud LoadBalancer 缓存

除了在每次必须选择实例时通过 DiscoveryClient 检索实例的基本 ServiceInstanceListSupplier 实现之外，我们还提供了两个缓存实现。

##### 3.3.1.Caffeine支持的 LoadBalancer 缓存实现

如果类路径中有 com.github.ben-manes.caffeine:caffeine，则将使用基于咖啡因的实现。有关如何配置它的信息，请参阅 LoadBalancerCacheConfiguration 部分。

如果您使用的是 Caffeine，您还可以通过在 spring.cloud.loadbalancer.cache.caffeine.spec 属性中传递您自己的 Caffeine Specification 来覆盖 LoadBalancer 的默认 Caffeine 缓存设置。

警告：传递您自己的 Caffeine 规范将覆盖任何其他 LoadBalancerCache 设置，包括常规 LoadBalancer 缓存配置字段，例如 ttl 和容量。

##### 3.3.2.默认 LoadBalancer 缓存实现

如果类路径中没有 Caffeine，则将使用 spring-cloud-starter-loadbalancer 自动附带的 DefaultLoadBalancerCache。有关如何配置它的信息，请参阅 LoadBalancerCacheConfiguration 部分。

要使用 Caffeine 而不是默认缓存，请将 com.github.ben-manes.caffeine:caffeine 依赖项添加到类路径。

##### 3.3.3.负载均衡器缓存配置

您可以设置自己的 ttl 值（写入后条目应过期的时间），表示为 Duration，方法是将符合 Spring Boot String 的 String 传递到 Duration 转换器语法。作为 spring.cloud.loadbalancer.cache.ttl 属性的值。您还可以通过设置 spring.cloud.loadbalancer.cache.capacity 属性的值来设置自己的 LoadBalancer 缓存初始容量。

默认设置包括 ttl 设置为 35 秒，默认 initialCapacity 为 256。

您还可以通过将 spring.cloud.loadbalancer.cache.enabled 的值设置为 false 来完全禁用 loadBalancer 缓存。

尽管基本的非缓存实现对于原型设计和测试很有用，但它的效率远低于缓存版本，因此我们建议始终在生产中使用缓存版本。如果缓存已由 DiscoveryClient 实现完成，例如 EurekaDiscoveryClient，则应禁用负载平衡器缓存以防止双重缓存。

#### 3.4.基于区域的负载平衡

为了启用基于区域的负载平衡，我们提供了 ZonePreferenceServiceInstanceListSupplier。我们使用 DiscoveryClient 特定的区域配置（例如，eureka.instance.metadata-map.zone）来选择客户端尝试过滤可用服务实例的区域。

您还可以通过设置 spring.cloud.loadbalancer.zone 属性的值来覆盖特定于 DiscoveryClient 的区域设置。

目前，只有 Eureka Discovery Client 被检测来设置 LoadBalancer 区域。对于其他发现客户端，设置 spring.cloud.loadbalancer.zone 属性。更多仪器即将推出。

为了确定检索到的 ServiceInstance 的区域，我们检查其元数据映射中“区域”键下的值。

ZonePreferenceServiceInstanceListSupplier 过滤检索到的实例并只返回同一区域内的实例。如果该区域为空或同一区域内没有实例，则返回所有检索到的实例。

为了使用基于区域的负载平衡方法，您必须在自定义配置中实例化 ZonePreferenceServiceInstanceListSupplier bean。

我们使用委托来处理 ServiceInstanceListSupplier bean。我们建议在 ZonePreferenceServiceInstanceListSupplier 的构造函数中传递一个 DiscoveryClientServiceInstanceListSupplier 委托，然后用 CachingServiceInstanceListSupplier 包装后者以利用 LoadBalancer 缓存机制。

您可以使用此示例配置进行设置：

```java
public class CustomLoadBalancerConfiguration {

    @Bean
    public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
                    .withDiscoveryClient()
                    .withZonePreference()
                    .withCaching()
                    .build(context);
    }
}
```

#### 3.5. LoadBalancer 的实例健康检查

可以为 LoadBalancer 启用计划的 HealthCheck。为此提供了 HealthCheckServiceInstanceListSupplier。它会定期验证委托 ServiceInstanceListSupplier 提供的实例是否仍然存在并且只返回健康的实例，除非没有 - 然后它返回所有检索到的实例。

这种机制在使用 SimpleDiscoveryClient 时特别有用。对于由实际 Service Registry 支持的客户端，没有必要使用，因为我们在查询外部 ServiceDiscovery 后已经获得了健康的实例。

还建议将此供应商用于每个服务具有少量实例的设置，以避免在失败的实例上重试调用。

如果使用任何服务发现支持的供应商，通常不需要添加此健康检查机制，因为我们直接从服务注册处检索实例的健康状态。

HealthCheckServiceInstanceListSupplier 依赖于由委托通量提供的更新实例。在极少数情况下，您想使用不刷新实例的委托，即使实例列表可能发生变化（例如我们提供的 DiscoveryClientServiceInstanceListSupplier），您可以设置 spring.cloud.loadbalancer.health-check.refetch -instances 为 true 以使 HealthCheckServiceInstanceListSupplier 刷新实例列表。然后，您还可以通过修改 spring.cloud.loadbalancer.health-check.refetch-instances-interval 的值来调整刷新间隔，并通过设置 spring.cloud.loadbalancer.health-check.repeat- 选择禁用额外的健康检查重复health-check 为 false，因为每个实例重新获取也会触发健康检查。

HealthCheckServiceInstanceListSupplier 使用以 spring.cloud.loadbalancer.health-check 为前缀的属性。您可以为调度程序设置 initialDelay 和间隔。您可以通过设置 spring.cloud.loadbalancer.health-check.path.default 属性的值来设置健康检查 URL 的默认路径。您还可以通过设置 spring.cloud.loadbalancer.health-check.path.[SERVICE_ID] 属性的值，将 [SERVICE_ID] 替换为您的服务的正确 ID，为任何给定服务设置特定值。如果未设置路径，则默认使用 /actuator/health。

如果您依赖默认路径 (/actuator/health)，请确保将 spring-boot-starter-actuator 添加到协作者的依赖项中，除非您计划自行添加此类端点。

为了使用健康检查调度程序方法，您必须在自定义配置中实例化 HealthCheckServiceInstanceListSupplier bean。

我们使用委托来处理 ServiceInstanceListSupplier bean。我们建议在 HealthCheckServiceInstanceListSupplier 的构造函数中传递一个 DiscoveryClientServiceInstanceListSupplier 委托。

您可以使用此示例配置进行设置：

```java
public class CustomLoadBalancerConfiguration {

    @Bean
    public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
                    .withDiscoveryClient()
                    .withHealthChecks()
                    .build(context);
        }
    }
```

对于非反应式堆栈，使用 withBlockingHealthChecks() 创建此供应商。您还可以传递您自己的 WebClient 或 RestTemplate 实例以用于检查。

HealthCheckServiceInstanceListSupplier 有自己的基于 Reactor Flux replay() 的缓存机制。因此，如果正在使用它，您可能希望跳过使用 CachingServiceInstanceListSupplier 包装该供应商。

#### 3.6. LoadBalancer 的相同实例首选项

您可以设置 LoadBalancer，使其更喜欢先前选择的实例（如果该实例可用）。

为此，您需要使用 SameInstancePreferenceServiceInstanceListSupplier。您可以通过将 spring.cloud.loadbalancer.configurations 的值设置为 same-instance-preference 或提供您自己的 ServiceInstanceListSupplier bean — 来配置它，例如：

```java
public class CustomLoadBalancerConfiguration {

    @Bean
    public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
                    .withDiscoveryClient()
                    .withSameInstancePreference()
                    .build(context);
        }
    }
```

这也是 Zookeeper StickyRule 的替代品。

#### 3.7. LoadBalancer 的基于请求的粘性会话

您可以设置 LoadBalancer，使其更喜欢在请求 cookie 中提供 instanceId 的实例。如果请求通过 ClientRequestContext 或 ServerHttpRequestContext 传递给 LoadBalancer，我们当前支持此功能，SC LoadBalancer 交换过滤器功能和过滤器使用它们。

为此，您需要使用 RequestBasedStickySessionServiceInstanceListSupplier。您可以通过将 spring.cloud.loadbalancer.configurations 的值设置为 request-based-sticky-session 或通过提供您自己的 ServiceInstanceListSupplier bean — 来配置它，例如：

```java
public class CustomLoadBalancerConfiguration {

    @Bean
    public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
                    .withDiscoveryClient()
                    .withRequestBasedStickySession()
                    .build(context);
        }
    }
```

对于该功能，在转发请求之前更新选定的服务实例（如果原始请求 cookie 不可用，则该实例可能与原始请求 cookie 中的服务实例不同）很有用。为此，请将 spring.cloud.loadbalancer.sticky-session.add-service-instance-cookie 的值设置为 true。

默认情况下，cookie 的名称是 sc-lb-instance-id。您可以通过更改 spring.cloud.loadbalancer.instance-id-cookie-name 属性的值来修改它。

WebClient 支持的负载平衡当前支持此功能。

#### 3.8. Spring Cloud LoadBalancer 提示

Spring Cloud LoadBalancer 允许您设置传递给 Request 对象内的 LoadBalancer 的字符串提示，稍后可以在可以处理它们的 ReactiveLoadBalancer 实现中使用。

您可以通过设置 spring.cloud.loadbalancer.hint.default 属性的值来为所有服务设置默认提示。您还可以通过设置 spring.cloud.loadbalancer.hint.[SERVICE_ID] 属性的值，将 [SERVICE_ID] 替换为您的服务的正确 ID，为任何给定服务设置特定值。如果用户未设置提示，则使用默认值。

#### 3.9.基于提示的负载平衡

我们还提供了一个 HintBasedServiceInstanceListSupplier，它是一个 ServiceInstanceListSupplier 实现，用于基于提示的实例选择。

HintBasedServiceInstanceListSupplier 检查提示请求标头（默认标头名称为 X-SC-LB-Hint，但您可以通过更改 spring.cloud.loadbalancer.hint-header-name 属性的值来修改它），如果是找到一个提示请求头，使用头中传递的提示值过滤服务实例。

如果没有添加提示头，HintBasedServiceInstanceListSupplier 使用属性中的提示值来过滤服务实例。

如果头或属性没有设置提示，则返回委托提供的所有服务实例。

在过滤时，HintBasedServiceInstanceListSupplier 查找在其 metadataMap 中的提示键下设置了匹配值的服务实例。如果没有找到匹配的实例，则返回委托提供的所有实例。

您可以使用以下示例配置进行设置：

```java
public class CustomLoadBalancerConfiguration {

    @Bean
    public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
                    .withDiscoveryClient()
                    .withHints()
                    .withCaching()
                    .build(context);
    }
}
```

#### 3.10.转换负载均衡的 HTTP 请求

您可以使用选定的 ServiceInstance 来转换负载均衡的 HTTP 请求。

对于 RestTemplate，需要实现和定义 LoadBalancerRequestTransformer 如下：

```java
@Bean
public LoadBalancerRequestTransformer transformer() {
    return new LoadBalancerRequestTransformer() {
        @Override
        public HttpRequest transformRequest(HttpRequest request, ServiceInstance instance) {
            return new HttpRequestWrapper(request) {
                @Override
                public HttpHeaders getHeaders() {
                    HttpHeaders headers = new HttpHeaders();
                    headers.putAll(super.getHeaders());
                    headers.add("X-InstanceId", instance.getInstanceId());
                    return headers;
                }
            };
        }
    };
}
```

对于WebClient，需要实现和定义LoadBalancerClientRequestTransformer如下：

```java
@Bean
public LoadBalancerClientRequestTransformer transformer() {
    return new LoadBalancerClientRequestTransformer() {
        @Override
        public ClientRequest transformRequest(ClientRequest request, ServiceInstance instance) {
            return ClientRequest.from(request)
                    .header("X-InstanceId", instance.getInstanceId())
                    .build();
        }
    };
}
```

如果定义了多个转换器，它们将按照定义 Bean 的顺序应用。或者，您可以使用 LoadBalancerRequestTransformer.DEFAULT_ORDER 或 LoadBalancerClientRequestTransformer.DEFAULT_ORDER 来指定顺序。

#### 3.11. Spring Cloud LoadBalancer 启动器

我们还提供了一个启动器，允许您在 Spring Boot 应用程序中轻松添加 Spring Cloud LoadBalancer。为了使用它，只需将 org.springframework.cloud:spring-cloud-starter-loadbalancer 添加到构建文件中的 Spring Cloud 依赖项中。

Spring Cloud LoadBalancer starter 包括 Spring Boot Caching 和 Evictor。

#### 3.12.传递你自己的 Spring Cloud LoadBalancer 配置

也可以使用@LoadBalancerClient注解传递自己的负载均衡客户端配置，传递负载均衡客户端的名称和配置类，如下：

```java
@Configuration
@LoadBalancerClient(value = "stores", configuration = CustomLoadBalancerConfiguration.class)
public class MyConfiguration {

    @Bean
    @LoadBalanced
    public WebClient.Builder loadBalancedWebClientBuilder() {
        return WebClient.builder();
    }
}
```

提示

为了更轻松地处理您自己的 LoadBalancer 配置，我们在 ServiceInstanceListSupplier 类中添加了 builder() 方法。

提示

您还可以使用我们的替代预定义配置代替默认配置，方法是将 spring.cloud.loadbalancer.configurations 属性的值设置为 zone-preference 以使用 ZonePreferenceServiceInstanceListSupplier 与缓存或健康检查以使用 HealthCheckServiceInstanceListSupplier 与缓存。

您可以使用此功能来实例化 ServiceInstanceListSupplier 或 ReactorLoadBalancer 的不同实现，它们可以由您编写，也可以由我们作为替代方案提供（例如 ZonePreferenceServiceInstanceListSupplier）以覆盖默认设置。

您可以在此处查看自定义配置示例。

注释值参数（存储在上面的示例中）指定了我们应该使用给定的自定义配置向其发送请求的服务的服务 ID。

您还可以通过 @LoadBalancerClients 注释传递多个配置（用于多个负载均衡器客户端），如以下示例所示：

```java
@Configuration
@LoadBalancerClients({@LoadBalancerClient(value = "stores", configuration = StoresLoadBalancerClientConfiguration.class), @LoadBalancerClient(value = "customers", configuration = CustomersLoadBalancerClientConfiguration.class)})
public class MyConfiguration {

    @Bean
    @LoadBalanced
    public WebClient.Builder loadBalancedWebClientBuilder() {
        return WebClient.builder();
    }
}
```

您作为 @LoadBalancerClient 或 @LoadBalancerClients 配置参数传递的类不应使用 @Configuration 进行注释或不在组件扫描范围内。

#### 3.13. Spring Cloud LoadBalancer 生命周期

使用自定义 LoadBalancer 配置注册可能有用的一种 bean 是 LoadBalancerLifecycle。

LoadBalancerLifecycle bean 提供回调方法，名为 onStart(Request<RC> request)、onStartRequest(Request<RC> request, Response<T> lbResponse) 和 onComplete(CompletionContext<RES, T, RC> completionContext)，您应该实现这些方法指定在负载平衡之前和之后应该执行的操作。

onStart(Request<RC> request) 将 Request 对象作为参数。它包含用于选择适当实例的数据，包括下游客户端请求和提示。 onStartRequest 还接受 Request 对象和 Response<T> 对象作为参数。另一方面，为 onComplete(CompletionContext<RES, T, RC> completionContext) 方法提供了一个 CompletionContext 对象。它包含 LoadBalancer 响应，包括选定的服务实例、针对该服务实例执行的请求的状态和（如果可用）返回到下游客户端的响应，以及（如果发生异常）相应的 Throwable。

supports(Class requestContextClass, Class responseClass, Class serverTypeClass) 方法可用于确定相关处理器是否处理所提供类型的对象。如果没有被用户覆盖，则返回 true。

上述方法调用中，RC表示RequestContext类型，RES表示客户端响应类型，T表示返回服务器类型。

#### 3.14. Spring Cloud LoadBalancer 统计

我们提供了一个名为 MicrometerStatsLoadBalancerLifecycle 的 LoadBalancerLifecycle bean，它使用 Micrometer 为负载平衡调用提供统计信息。

为了将此 bean 添加到您的应用程序上下文中，请将 spring.cloud.loadbalancer.stats.micrometer.enabled 的值设置为 true 并使用 MeterRegistry（例如，通过将 Spring Boot Actuator 添加到您的项目中）。

MicrometerStatsLoadBalancerLifecycle 在 MeterRegistry 中注册以下仪表：

- loadbalancer.requests.active：允许您监控任何服务实例的当前活动请求数量的量表（服务实例数据可通过标签获得）； 
- loadbalancer.requests.success：一个计时器，用于测量已结束将响应传递给底层客户端的任何负载平衡请求的执行时间； 
- loadbalancer.requests.failed：一个计时器，用于测量任何以异常结束的负载平衡请求的执行时间； 
- loadbalancer.requests.discard：一个计数器，用于测量被丢弃的负载平衡请求的数量，即负载均衡器尚未检索到运行请求的服务实例的请求。

只要可用，有关服务实例、请求数据和响应数据的附加信息就会通过标签添加到指标中。

对于某些实现，例如 BlockingLoadBalancerClient，请求和响应数据可能不可用，因为我们从参数建立泛型类型并且可能无法确定类型并读取数据。

当为给定仪表添加至少一个记录时，仪表将在注册表中注册。

您可以通过添加 MeterFilters 进一步配置这些指标的行为（例如，添加发布百分位数和直方图）

### 4. Spring Cloud 断路器

#### 4.1.介绍

Spring Cloud 断路器提供了跨不同断路器实现的抽象。它提供了在您的应用程序中使用的一致 API，让您（开发人员）可以选择最适合您的应用程序需求的断路器实现。

##### 4.1.1.支持的实现

Spring Cloud 支持以下断路器实现：

- [Resilience4J](https://github.com/resilience4j/resilience4j)
- [Sentinel](https://github.com/alibaba/Sentinel)
- [Spring Retry](https://github.com/spring-projects/spring-retry)

#### 4.2.核心概念

要在您的代码中创建断路器，您可以使用 CircuitBreakerFactory API。当您在类路径中包含 Spring Cloud Circuit Breaker starter 时，会自动为您创建一个实现此 API 的 bean。以下示例显示了如何使用此 API 的简单示例：

```java
@Service
public static class DemoControllerService {
    private RestTemplate rest;
    private CircuitBreakerFactory cbFactory;

    public DemoControllerService(RestTemplate rest, CircuitBreakerFactory cbFactory) {
        this.rest = rest;
        this.cbFactory = cbFactory;
    }

    public String slow() {
        return cbFactory.create("slow").run(() -> rest.getForObject("/slow", String.class), throwable -> "fallback");
    }

}
```

CircuitBreakerFactory.create API 创建了一个名为 CircuitBreaker 的类的实例。 run 方法接受一个供应商和一个函数。供应商是您要包装在断路器中的代码。该功能是在断路器跳闸时运行的回退。该函数传递了引发回退的 Throwable。如果您不想提供后备，您可以选择排除后备。

##### 4.2.1.反应式代码中的断路器

如果 Project Reactor 在类路径上，您还可以将 ReactiveCircuitBreakerFactory 用于您的反应式代码。以下示例显示了如何执行此操作：

```java
@Service
public static class DemoControllerService {
    private ReactiveCircuitBreakerFactory cbFactory;
    private WebClient webClient;


    public DemoControllerService(WebClient webClient, ReactiveCircuitBreakerFactory cbFactory) {
        this.webClient = webClient;
        this.cbFactory = cbFactory;
    }

    public Mono<String> slow() {
        return webClient.get().uri("/slow").retrieve().bodyToMono(String.class).transform(
        it -> cbFactory.create("slow").run(it, throwable -> return Mono.just("fallback")));
    }
}
```

ReactiveCircuitBreakerFactory.create API 创建了一个名为 ReactiveCircuitBreaker 的类的实例。 run 方法采用 Mono 或 Flux 并将其包装在断路器中。您可以选择配置回退函数，如果断路器跳闸并传递导致故障的 Throwable，则将调用该函数。

#### 4.3.配置

您可以通过创建自定义程序类型的 bean 来配置断路器。定制器接口有一个方法（称为定制），可以让对象进行定制。

有关如何自定义给定实现的详细信息，请参阅以下文档：

- [Resilience4J](https://docs.spring.io/spring-cloud-commons/spring-cloud-circuitbreaker/current/reference/html/spring-cloud-circuitbreaker.html#configuring-resilience4j-circuit-breakers)
- [Sentinal](https://github.com/alibaba/spring-cloud-alibaba/blob/master/spring-cloud-alibaba-docs/src/main/asciidoc/circuitbreaker-sentinel.adoc#circuit-breaker-spring-cloud-circuit-breaker-with-sentinel—configuring-sentinel-circuit-breakers)
- [Spring Retry](https://docs.spring.io/spring-cloud-circuitbreaker/docs/current/reference/html/spring-cloud-circuitbreaker.html#configuring-spring-retry-circuit-breakers)

每次调用 CircuitBreaker#run 时，某些 CircuitBreaker 实现（例如 Resilience4JCircuitBreaker）都会调用自定义方法。它可能效率低下。在这种情况下，您可以使用 CircuitBreaker#once 方法。在多次调用自定义没有意义的情况下很有用，例如，在使用 Resilience4j 的事件的情况下。

下面的例子展示了每个 io.github.resilience4j.circuitbreaker.CircuitBreaker 消费事件的方式。

```java
Customizer.once(circuitBreaker -> {
  circuitBreaker.getEventPublisher()
    .onStateTransition(event -> log.info("{}: {}", event.getCircuitBreakerName(), event.getStateTransition()));
}, CircuitBreaker::getName)
```

### 5. CachedRandomPropertySource

Spring Cloud Context 提供了一个 PropertySource，它根据一个键缓存随机值。在缓存功能之外，它的工作方式与 Spring Boot 的 RandomValuePropertySource 相同。如果您想要一个即使在 Spring 应用程序上下文重新启动后也保持一致的随机值，则此随机值可能很有用。属性值采用 cachedrandom.[yourkey].[type] 的形式，其中 yourkey 是缓存中的键。类型值可以是 Spring Boot 的 RandomValuePropertySource 支持的任何类型。

```properties
myrandom=${cachedrandom.appname.value}
```

### [6. Security](https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/#spring-cloud-security)

#### [ 6.1. Single Sign On](https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/#spring-cloud-security-single-sign-on)

所有 OAuth2 SSO 和资源服务器功能都在 1.3 版中移至 Spring Boot。您可以在 Spring Boot 用户指南中找到文档。

##### 6.1.1.客户端令牌中继relay

如果您的应用是面向 OAuth2 客户端的用户（即已声明 @EnableOAuth2Sso 或 @EnableOAuth2Client），则它在 Spring Boot 的请求范围内具有 OAuth2ClientContext。您可以从此上下文创建自己的 OAuth2RestTemplate 和自动装配的 OAuth2ProtectedResourceDetails，然后上下文将始终向下游转发访问令牌，并在访问令牌过期时自动刷新访问令牌。 （这些是 Spring Security 和 Spring Boot 的特性。）

##### 6.1.2.资源服务器令牌中继relay

如果您的应用程序具有 @EnableResourceServer，您可能希望将传入的令牌向下游中继到其他服务。如果您使用 RestTemplate 来联系下游服务，那么这只是如何使用正确的上下文创建模板的问题。

如果您的服务使用 UserInfoTokenServices 来验证传入的令牌（即它使用 security.oauth2.user-info-uri 配置），那么您可以简单地使用自动装配的 OAuth2ClientContext 创建一个 OAuth2RestTemplate（它将在命中之前由身份验证过程填充后端代码）。等效地（使用 Spring Boot 1.4），您可以注入 UserInfoRestTemplateFactory 并在您的配置中获取其 OAuth2RestTemplate。例如：

**MyConfiguration.java**

```java
@Bean
public OAuth2RestTemplate restTemplate(UserInfoRestTemplateFactory factory) {
    return factory.getUserInfoRestTemplate();
}
```

这个 rest 模板将具有与身份验证过滤器使用的相同的 OAuth2ClientContext（请求范围），因此您可以使用它来发送具有相同访问令牌的请求。

如果您的应用程序没有使用 UserInfoTokenServices 但仍然是客户端（即它声明了 @EnableOAuth2Client 或 @EnableOAuth2Sso），那么使用 Spring Security Cloud，用户从 @Autowired OAuth2Context 创建的任何 OAuth2RestOperations 也将转发令牌。这个特性默认实现为一个MVC处理程序拦截器，所以它只适用于Spring MVC。如果您不使用 MVC，您可以使用自定义过滤器或 AOP 拦截器包装 AccessTokenContextRelay 来提供相同的功能。

这是一个基本示例，展示了使用在别处创建的自动装配的休息模板（“foo.com”是一个资源服务器，接受与周围应用程序相同的令牌）：

**MyController.java**

```java
@Autowired
private OAuth2RestOperations restTemplate;

@RequestMapping("/relay")
public String relay() {
    ResponseEntity<String> response =
      restTemplate.getForEntity("https://foo.com/bar", String.class);
    return "Success! (" + response.getBody() + ")";
}
```

如果您不想转发令牌（这是一个有效的选择，因为您可能想扮演自己的角色，而不是向您发送令牌的客户端），那么您只需要创建自己的 OAuth2Context 而不是自动装配默认一个。

如果可用，Feign 客户端还将选择使用 OAuth2ClientContext 的拦截器，因此他们还应该在 RestTemplate 所在的任何地方进行令牌中继。

### 7. 配置属性

附录 A：常见的应用程序属性

可以在 application.properties 文件、application.yml 文件或命令行开关中指定各种属性。本附录提供了常见 Spring Cloud Commons 属性的列表以及对使用它们的底层类的引用。

属性贡献可以来自类路径上的其他 jar 文件，因此您不应认为这是一个详尽的列表。此外，您可以定义自己的属性。

| 名称                                                         | 默认值              | 描述                                                         |
| :----------------------------------------------------------- | :------------------ | :----------------------------------------------------------- |
| spring.cloud.compatibility-verifier.compatible-boot-versions |                     | Spring Boot 依赖项的默认接受版本。如果不想指定具体值，可以为补丁版本设置 {@code x}。示例：{@code 3.4.x} |
| spring.cloud.compatibility-verifier.enabled                  | `false`             | 启用创建 Spring Cloud 兼容性验证。                           |
| spring.cloud.config.allow-override                           | `true`              | 指示可以使用 {@link #isOverrideSystemProperties() systemPropertiesOverride} 的标志。设置为 false 以防止用户意外更改默认值。默认为真。 |
| spring.cloud.config.override-none                            | `false`             | 标记以指示当 {@link #setAllowOverride(boolean) allowOverride} 为 true 时，外部属性应具有最低优先级并且不应覆盖任何现有属性源（包括本地配置文件）。默认为假。 |
| spring.cloud.config.override-system-properties               | `true`              | 标志以指示外部属性应覆盖系统属性。默认为真。                 |
| spring.cloud.decrypt-environment-post-processor.enabled      | `true`              | 启用 DecryptEnvironmentPostProcessor。                       |
| spring.cloud.discovery.client.composite-indicator.enabled    | `true`              | 启用发现客户端复合健康指标。                                 |
| spring.cloud.discovery.client.health-indicator.enabled       | `true`              |                                                              |
| spring.cloud.discovery.client.health-indicator.include-description | `false`             |                                                              |
| spring.cloud.discovery.client.health-indicator.use-services-query | `true`              | 指标是否应使用 {@link DiscoveryClient#getServices} 来检查其健康状况。当设置为 {@code false} 时，指示器会使用较轻的 {@link DiscoveryClient#probe()}。这在返回的服务数量使操作不必要地繁重的大型部署中很有帮助。 |
| spring.cloud.discovery.client.simple.instances               |                     |                                                              |
| spring.cloud.discovery.client.simple.order                   |                     |                                                              |
| spring.cloud.discovery.enabled                               | `true`              | 启用发现客户端健康指标。                                     |
| spring.cloud.features.enabled                                | `true`              | 启用功能端点。                                               |
| spring.cloud.httpclientfactories.apache.enabled              | `true`              | 允许创建 Apache Http 客户端工厂 bean。                       |
| spring.cloud.httpclientfactories.ok.enabled                  | `true`              | 启用 OK Http Client 工厂 bean 的创建。                       |
| spring.cloud.hypermedia.refresh.fixed-delay                  | `5000`              |                                                              |
| spring.cloud.hypermedia.refresh.initial-delay                | `10000`             |                                                              |
| spring.cloud.inetutils.default-hostname                      | `localhost`         | 默认主机名。发生错误时使用。                                 |
| spring.cloud.inetutils.default-ip-address                    | `127.0.0.1`         | 默认 IP 地址。发生错误时使用。                               |
| spring.cloud.inetutils.ignored-interfaces                    |                     | 将被忽略的网络接口的 Java 正则表达式列表。                   |
| spring.cloud.inetutils.preferred-networks                    |                     | 首选网络地址的 Java 正则表达式列表。                         |
| spring.cloud.inetutils.timeout-seconds                       | `1`                 | 计算主机名的超时时间，以秒为单位。                           |
| spring.cloud.inetutils.use-only-site-local-interfaces        | `false`             | 是否仅使用具有站点本地地址的接口。有关更多详细信息，请参阅 {@link InetAddress#isSiteLocalAddress()}。 |
| spring.cloud.loadbalancer.cache.caffeine.spec                |                     | 用于创建缓存的规范。有关规范格式的更多详细信息，请参阅 CaffeineSpec。 |
| spring.cloud.loadbalancer.cache.capacity                     | `256`               | 初始缓存容量表示为 int。                                     |
| spring.cloud.loadbalancer.cache.enabled                      | `true`              | 启用 Spring Cloud LoadBalancer 缓存机制                      |
| spring.cloud.loadbalancer.cache.ttl                          | `35s`               | 生存时间 - 从写入记录开始计算的时间，之后缓存条目过期，表示为 {@link Duration}。属性 {@link String} 必须符合 Spring Boot <code>StringToDurationConverter</code> 中指定的适当语法。 @see <a href="https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot/src/main/java/org/springframework/boot/convert /StringToDurationConverter.java">StringToDurationConverter.java</a> |
| spring.cloud.loadbalancer.configurations                     | `default`           | 启用预定义的 LoadBalancer 配置。                             |
| spring.cloud.loadbalancer.enabled                            | `true`              | 启用 Spring Cloud LoadBalancer。                             |
| spring.cloud.loadbalancer.health-check.initial-delay         | `0`                 | HealthCheck 调度程序的初始延迟值。                           |
| spring.cloud.loadbalancer.health-check.interval              | `25s`               | 重新运行 HealthCheck 调度程序的时间间隔。                    |
| spring.cloud.loadbalancer.health-check.path                  |                     |                                                              |
| spring.cloud.loadbalancer.health-check.refetch-instances     | `false`             | 指示是否应由 <code>HealthCheckServiceInstanceListSupplier</code> 重新获取实例。如果实例可以更新并且底层委托不提供持续的流量，则可以使用此方法。 |
| spring.cloud.loadbalancer.health-check.refetch-instances-interval | `25s`               | 重新获取可用服务实例的时间间隔。                             |
| spring.cloud.loadbalancer.health-check.repeat-health-check   | `true`              | 指示是否应继续重复运行状况检查。如果定期重新获取实例，将其设置为 <code>false</code> 可能会很有用，因为每次重新获取也会触发健康检查。 |
| spring.cloud.loadbalancer.hint                               |                     | 允许设置传递给 LoadBalancer 请求的 <code>hint</code> 值，随后可以在 {@link ReactiveLoadBalancer} 实现中使用。 |
| spring.cloud.loadbalancer.hint-header-name                   | `X-SC-LB-Hint`      | 允许设置用于传递基于提示的服务实例过滤提示的标头名称。       |
| spring.cloud.loadbalancer.retry.avoid-previous-instance      | `true`              | 如果 Spring-Retry 在类路径中，则启用使用 RetryAwareServiceInstanceListSupplier 包装 ServiceInstanceListSupplier bean。 |
| spring.cloud.loadbalancer.retry.backoff.enabled              | `false`             | 指示是否应应用反应器重试退避。                               |
| spring.cloud.loadbalancer.retry.backoff.jitter               | `0.5`               | 用于设置 {@link RetryBackoffSpec#jitter}。                   |
| spring.cloud.loadbalancer.retry.backoff.max-backoff          |                     | 用于设置 {@link RetryBackoffSpec#maxBackoff}。               |
| spring.cloud.loadbalancer.retry.backoff.min-backoff          | `5ms`               | 用于设置 {@link RetryBackoffSpec#minBackoff}。               |
| spring.cloud.loadbalancer.retry.enabled                      | `true`              | 启用 LoadBalancer 重试。                                     |
| spring.cloud.loadbalancer.retry.max-retries-on-next-service-instance | `1`                 | 要在下一个 <code>ServiceInstance</code> 上执行的重试次数。在每次重试调用之前选择一个 <code>ServiceInstance</code>。 |
| spring.cloud.loadbalancer.retry.max-retries-on-same-service-instance | `0`                 | 要在同一 <code>ServiceInstance</code> 上执行的重试次数。     |
| spring.cloud.loadbalancer.retry.retry-on-all-operations      | `false`             | 表示应尝试对除 {@link HttpMethod#GET} 以外的操作进行重试。   |
| spring.cloud.loadbalancer.retry.retryable-status-codes       |                     | 应触发重试的状态代码 {@link Set}。                           |
| spring.cloud.loadbalancer.service-discovery.timeout          |                     | 调用服务发现的超时时间的字符串表示形式。                     |
| spring.cloud.loadbalancer.sticky-session.add-service-instance-cookie | `false`             | 指示 SC LoadBalancer 是否应添加带有新选择实例的 cookie。     |
| spring.cloud.loadbalancer.sticky-session.instance-id-cookie-name | `sc-lb-instance-id` | 保存首选实例 ID 的 cookie 的名称。                           |
| spring.cloud.loadbalancer.zone                               |                     | Spring Cloud LoadBalancer 区域。                             |
| spring.cloud.refresh.additional-property-sources-to-retain   |                     | 刷新期间要保留的其他属性源。通常只保留系统属性源。此属性还允许保留属性源，例如由 EnvironmentPostProcessors 创建的属性源。 |
| spring.cloud.refresh.enabled                                 | `true`              | 启用刷新范围和相关功能的自动配置。                           |
| spring.cloud.refresh.extra-refreshable                       | `true`              | 用于将处理后处理到刷新范围的 bean 的其他类名。               |
| spring.cloud.refresh.never-refreshable                       | `true`              | 逗号分隔的 bean 类名列表，永远不会被刷新或反弹。             |
| spring.cloud.service-registry.auto-registration.enabled      | `true`              | 是否开启服务自动注册。默认为真。                             |
| spring.cloud.service-registry.auto-registration.fail-fast    | `false`             | 如果没有 AutoServiceRegistration，是否启动失败。默认为假。   |
| spring.cloud.service-registry.auto-registration.register-management | `true`              | 是否将管理注册为服务。默认为真。                             |
| spring.cloud.util.enabled                                    | `true`              | 启用创建 Spring Cloud 实用程序 bean。                        |