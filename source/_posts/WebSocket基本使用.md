---
title: WebSocket基本使用
date: 2021-11-09 08:12:43
tags: [网络协议, websocket]
categories: [网络协议, websocket]
---


# WebSocket基本使用

## WebSockets

文档涵盖了对 Servlet 堆栈的支持、包含原始 WebSocket 交互的 WebSocket 消息传递、通过 SockJS 的 WebSocket 模拟以及通过作为 WebSocket 子协议的 STOMP 的发布订阅消息传递。

### WebSocket 简介

WebSocket 协议 RFC 6455 提供了一种标准化方法，可通过单个 TCP 连接在客户端和服务器之间建立全双工、双向通信通道。它是与 HTTP 不同的 TCP 协议，但旨在通过 HTTP 工作，使用端口 80 和 443，并允许重新使用现有的防火墙规则。

WebSocket 交互以 HTTP 请求开始，该请求使用 HTTP Upgrade header进行升级，或者在本例中切换到 WebSocket 协议。以下示例显示了这样的交互：

```yaml
GET /spring-websocket-portfolio/portfolio HTTP/1.1
Host: localhost:8080
Upgrade: websocket 
Connection: Upgrade 
Sec-WebSocket-Key: Uc9l9TMkWGbHFD2qnFHltg==
Sec-WebSocket-Protocol: v10.stomp, v11.stomp
Sec-WebSocket-Version: 13
Origin: http://localhost:8080
```

与通常的 200 状态代码不同，具有 WebSocket 支持的服务器返回类似于以下内容的输出：

```yaml
HTTP/1.1 101 Switching Protocols 
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: 1qVdfYHU9hPOl4JYYNXF623Gzn0=
Sec-WebSocket-Protocol: v10.stomp
```

成功握手后，HTTP 升级请求底层的 TCP 套接字保持打开状态，客户端和服务器都可以继续发送和接收消息。

对 WebSockets 工作原理的完整介绍超出了本文档的范围。请参阅 RFC 6455、HTML5 的 WebSocket 章节或 Web 上的许多介绍和教程中的任何一个。

请注意，如果 WebSocket 服务器在 Web 服务器（例如 nginx）后面运行，您可能需要将其配置为将 WebSocket 升级请求传递到 WebSocket 服务器。同样，如果应用程序在云环境中运行，请查看与 WebSocket 支持相关的云提供商的说明。

#### HTTP 与 WebSocket

尽管 WebSocket 被设计为与 HTTP 兼容并从 HTTP 请求开始，但重要的是要了解这两种协议会导致非常不同的架构和应用程序编程模型。

在 HTTP 和 REST 中，一个应用程序被建模为多个 URL。为了与应用程序交互，客户端访问这些 URL，请求-响应样式。服务器根据 HTTP URL、方法和标头将请求路由到适当的处理程序。

相比之下，在 WebSockets 中，通常只有一个 URL 用于初始连接。随后，所有应用程序消息都在同一个 TCP 连接上流动。这指向一个完全不同的异步、事件驱动、消息传递架构。

WebSocket 也是一种低级传输协议，与 HTTP 不同，它不对消息内容规定任何语义。这意味着除非客户端和服务器就消息语义达成一致，否则无法路由或处理消息。

WebSocket 客户端和服务器可以通过 HTTP 握手请求上的 Sec-WebSocket-Protocol 标头协商使用更高级别的消息传递协议（例如，STOMP）。如果没有，他们需要提出自己的约定。

#### 何时使用 WebSocket

WebSockets 可以使网页具有动态性和交互性。但是，在许多情况下，Ajax 和 HTTP 流或长轮询的组合可以提供简单有效的解决方案。

例如，新闻、邮件和社交提要需要动态更新，但每隔几分钟更新一次可能完全没问题。另一方面，协作、游戏和金融应用程序需要更接近实时。

延迟本身并不是决定性因素。如果消息量相对较低（例如监控网络故障），HTTP 流或轮询可以提供有效的解决方案。低延迟、高频率和高容量的组合是使用 WebSocket 的最佳案例。

还请记住，在 Internet 上，不受您控制的限制性代理可能会阻止 WebSocket 交互，因为它们未配置为传递 Upgrade 标头，或者因为它们关闭了看似空闲的长期连接。这意味着将 WebSocket 用于防火墙内的内部应用程序是一个比面向公众的应用程序更直接的决定。

### WebSocket API

Spring Framework 提供了一个 WebSocket API，您可以使用它来编写处理 WebSocket 消息的客户端和服务器端应用程序。

#### WebSocketHandler

创建 WebSocket 服务器就像实现 WebSocketHandler 一样简单，或者更有可能扩展 TextWebSocketHandler 或 BinaryWebSocketHandler。以下示例使用 TextWebSocketHandler：

```java
import org.springframework.web.socket.WebSocketHandler;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.TextMessage;

public class MyHandler extends TextWebSocketHandler {

    @Override
    public void handleTextMessage(WebSocketSession session, TextMessage message) {
        // ...
    }

}
```

有专用的 WebSocket Java 配置和 XML 命名空间支持，用于将前面的 WebSocket 处理程序映射到特定 URL，如以下示例所示：

```java
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.config.annotation.WebSocketConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketHandlerRegistry;

@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(myHandler(), "/myHandler");
    }

    @Bean
    public WebSocketHandler myHandler() {
        return new MyHandler();
    }

}
```

以下示例显示了与前面示例等效的 XML 配置：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:websocket="http://www.springframework.org/schema/websocket"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/websocket
        https://www.springframework.org/schema/websocket/spring-websocket.xsd">

    <websocket:handlers>
        <websocket:mapping path="/myHandler" handler="myHandler"/>
    </websocket:handlers>

    <bean id="myHandler" class="org.springframework.samples.MyHandler"/>

</beans>
```

前面的示例用于 Spring MVC 应用程序，应包含在 DispatcherServlet 的配置中。但是，Spring 的 WebSocket 支持不依赖于 Spring MVC。在 WebSocketHttpRequestHandler 的帮助下，将 WebSocketHandler 集成到其他 HTTP 服务环境中相对简单。

直接与间接使用 WebSocketHandler API 时，例如通过 STOMP 消息传递，应用程序必须同步消息的发送，因为底层标准 WebSocket 会话 (JSR-356) 不允许并发发送。一种选择是使用 ConcurrentWebSocketSessionDecorator 包装 WebSocketSession。

#### WebSocket 握手

自定义初始 HTTP WebSocket 握手请求的最简单方法是通过 HandshakeInterceptor，它公开握手“之前”和“之后”的方法。您可以使用此类拦截器来阻止握手或使任何属性可用于 WebSocketSession。以下示例使用内置拦截器将 HTTP 会话属性传递给 WebSocket 会话：

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(new MyHandler(), "/myHandler")
            .addInterceptors(new HttpSessionHandshakeInterceptor());
    }

}
```

以下示例显示了与前面示例等效的 XML 配置：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:websocket="http://www.springframework.org/schema/websocket"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/websocket
        https://www.springframework.org/schema/websocket/spring-websocket.xsd">

    <websocket:handlers>
        <websocket:mapping path="/myHandler" handler="myHandler"/>
        <websocket:handshake-interceptors>
            <bean class="org.springframework.web.socket.server.support.HttpSessionHandshakeInterceptor"/>
        </websocket:handshake-interceptors>
    </websocket:handlers>

    <bean id="myHandler" class="org.springframework.samples.MyHandler"/>

</beans>
```

一个更高级的选项是扩展 DefaultHandshakeHandler，它执行 WebSocket 握手的步骤，包括验证客户端来源、协商子协议和其他细节。如果应用程序需要配置自定义 RequestUpgradeStrategy 以适应尚不支持的 WebSocket 服务器引擎和版本，则它也可能需要使用此选项（有关此主题的更多信息，请参阅部署）。 Java 配置和 XML 命名空间都可以配置自定义 HandshakeHandler。

Spring 提供了一个 WebSocketHandlerDecorator 基类，您可以使用它来装饰具有附加行为的 WebSocketHandler。使用 WebSocket Java 配置或 XML 命名空间时，默认提供并添加日志记录和异常处理实现。 ExceptionWebSocketHandlerDecorator 捕获任何 WebSocketHandler 方法产生的所有未捕获的异常，并关闭状态为 1011 的 WebSocket 会话，这表示服务器错误。

#### Deployment

Spring WebSocket API 很容易集成到 Spring MVC 应用程序中，其中 DispatcherServlet 为 HTTP WebSocket 握手和其他 HTTP 请求提供服务。通过调用 WebSocketHttpRequestHandler 也很容易集成到其他 HTTP 处理场景中。这很方便，也很容易理解。但是，对于 JSR-356 运行时需要特殊考虑。

Java WebSocket API (JSR-356) 提供了两种部署机制。第一个涉及启动时的 Servlet 容器类路径扫描（Servlet 3 特性）。另一个是在 Servlet 容器初始化时使用的注册 API。这两种机制都无法使用单个“前端控制器”进行所有 HTTP 处理 — 包括 WebSocket 握手和所有其他 HTTP 请求 — ，例如 Spring MVC 的 DispatcherServlet。

这是 JSR-356 的一个重大限制，即使在 JSR-356 运行时中运行时，Spring 的 WebSocket 支持也可以解决特定于服务器的 RequestUpgradeStrategy 实现。 Tomcat、Jetty、GlassFish、WebLogic、WebSphere 和 Undertow（以及 WildFly）目前存在此类策略。

已经创建了克服 Java WebSocket API 中上述限制的请求，可以在 eclipse-ee4j/websocket-api#211 中遵循该请求。 Tomcat、Undertow 和 WebSphere 提供了它们自己的 API 替代方案，可以做到这一点，Jetty 也可以做到。我们希望更多的服务器能做同样的事情。

第二个考虑因素是，支持 JSR-356 的 Servlet 容器预计会执行 ServletContainerInitializer (SCI) 扫描，这会减慢应用程序启动速度 — 在某些情况下会显着降低。如果在升级到支持 JSR-356 的 Servlet 容器版本后观察到显着影响，应该可以通过使用 web 中的 <absolute-ordering /> 元素有选择地启用或禁用 web 片段（和 SCI 扫描） .xml，如以下示例所示：

```xml
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://java.sun.com/xml/ns/javaee
        https://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
    version="3.0">

    <absolute-ordering/>

</web-app>
```

然后，您可以按名称有选择地启用 Web 片段，例如 Spring 自己的 SpringServletContainerInitializer，它提供对 Servlet 3 Java 初始化 API 的支持。以下示例显示了如何执行此操作：

```xml
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://java.sun.com/xml/ns/javaee
        https://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
    version="3.0">

    <absolute-ordering>
        <name>spring_web</name>
    </absolute-ordering>

</web-app>
```

#### 服务配置

每个底层 WebSocket 引擎都公开控制运行时特性的配置属性，例如消息缓冲区大小、空闲超时等。

对于 Tomcat、WildFly 和 GlassFish，您可以将 ServletServerContainerFactoryBean 添加到您的 WebSocket Java 配置中，如以下示例所示：

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Bean
    public ServletServerContainerFactoryBean createWebSocketContainer() {
        ServletServerContainerFactoryBean container = new ServletServerContainerFactoryBean();
        container.setMaxTextMessageBufferSize(8192);
        container.setMaxBinaryMessageBufferSize(8192);
        return container;
    }

}
```

以下示例显示了与前面示例等效的 XML 配置：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:websocket="http://www.springframework.org/schema/websocket"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/websocket
        https://www.springframework.org/schema/websocket/spring-websocket.xsd">

    <bean class="org.springframework...ServletServerContainerFactoryBean">
        <property name="maxTextMessageBufferSize" value="8192"/>
        <property name="maxBinaryMessageBufferSize" value="8192"/>
    </bean>

</beans>
```

对于客户端 WebSocket 配置，您应该使用 WebSocketContainerFactoryBean (XML) 或 ContainerProvider.getWebSocketContainer()（Java 配置）。

对于 Jetty，您需要提供一个预配置的 Jetty WebSocketServerFactory 并通过 WebSocket Java 配置将其插入 Spring 的 DefaultHandshakeHandler。以下示例显示了如何执行此操作：

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(echoWebSocketHandler(),
            "/echo").setHandshakeHandler(handshakeHandler());
    }

    @Bean
    public DefaultHandshakeHandler handshakeHandler() {

        WebSocketPolicy policy = new WebSocketPolicy(WebSocketBehavior.SERVER);
        policy.setInputBufferSize(8192);
        policy.setIdleTimeout(600000);

        return new DefaultHandshakeHandler(
                new JettyRequestUpgradeStrategy(new WebSocketServerFactory(policy)));
    }

}
```

以下示例显示了与前面示例等效的 XML 配置：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:websocket="http://www.springframework.org/schema/websocket"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/websocket
        https://www.springframework.org/schema/websocket/spring-websocket.xsd">

    <websocket:handlers>
        <websocket:mapping path="/echo" handler="echoHandler"/>
        <websocket:handshake-handler ref="handshakeHandler"/>
    </websocket:handlers>

    <bean id="handshakeHandler" class="org.springframework...DefaultHandshakeHandler">
        <constructor-arg ref="upgradeStrategy"/>
    </bean>

    <bean id="upgradeStrategy" class="org.springframework...JettyRequestUpgradeStrategy">
        <constructor-arg ref="serverFactory"/>
    </bean>

    <bean id="serverFactory" class="org.eclipse.jetty...WebSocketServerFactory">
        <constructor-arg>
            <bean class="org.eclipse.jetty...WebSocketPolicy">
                <constructor-arg value="SERVER"/>
                <property name="inputBufferSize" value="8092"/>
                <property name="idleTimeout" value="600000"/>
            </bean>
        </constructor-arg>
    </bean>

</beans>
```

#### Allowed Origins

从 Spring Framework 4.1.5 开始，WebSocket 和 SockJS 的默认行为是仅接受同源请求。还可以允许所有或指定的来源列表。此检查主要是为浏览器客户端设计的。没有什么可以阻止其他类型的客户端修改 Origin 标头值（有关更多详细信息，请参阅 RFC 6454：Web Origin 概念）。

三种可能的行为是：

- 仅允许同源请求（默认）：在这种模式下，当启用 SockJS 时，Iframe HTTP 响应头 X-Frame-Options 设置为 SAMEORIGIN，并且禁用 JSONP 传输，因为它不允许检查源要求。因此，启用此模式时不支持 IE6 和 IE7。
- 允许指定的来源列表：每个允许的来源必须以 http:// 或 https:// 开头。在这种模式下，当 SockJS 启用时，IFrame 传输被禁用。因此，启用此模式时，不支持 IE6 到 IE9。
- 允许所有来源：要启用此模式，您应该提供 * 作为允许的来源值。在这种模式下，所有传输都可用。

您可以配置 WebSocket 和 SockJS 允许的来源，如以下示例所示：

```java
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.config.annotation.WebSocketConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketHandlerRegistry;

@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(myHandler(), "/myHandler").setAllowedOrigins("https://mydomain.com");
    }

    @Bean
    public WebSocketHandler myHandler() {
        return new MyHandler();
    }

}
```

以下示例显示了与前面示例等效的 XML 配置：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:websocket="http://www.springframework.org/schema/websocket"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/websocket
        https://www.springframework.org/schema/websocket/spring-websocket.xsd">

    <websocket:handlers allowed-origins="https://mydomain.com">
        <websocket:mapping path="/myHandler" handler="myHandler" />
    </websocket:handlers>

    <bean id="myHandler" class="org.springframework.samples.MyHandler"/>

</beans>
```

### SockJS Fallback

在公共 Internet 上，不受您控制的限制性代理可能会阻止 WebSocket 交互，因为它们未配置为传递 Upgrade 标头，或者因为它们关闭了看似空闲的长期连接。

该问题的解决方案是 WebSocket 模拟 — 即尝试首先使用 WebSocket，然后再使用基于 HTTP 的技术来模拟 WebSocket 交互并公开相同的应用程序级 API。

在 Servlet 堆栈上，Spring Framework 为 SockJS 协议提供服务器（和客户端）支持。

#### 概述

SockJS 的目标是让应用程序使用 WebSocket API，但在运行时必要时回退到非 WebSocket 替代方案，而无需更改应用程序代码。

SockJS 包括：

- 以可执行叙述测试的形式定义的 SockJS 协议。
- SockJS JavaScript 客户端 — 用于浏览器的客户端库。
- SockJS 服务器实现，包括 Spring Framework spring websocket 模块中的一个。
- spring-websocket 模块中的 SockJS Java 客户端（自 4.1 版起）。

SockJS 是为在浏览器中使用而设计的。它使用多种技术来支持广泛的浏览器版本。有关 SockJS 传输类型和浏览器的完整列表，请参阅 SockJS 客户端页面。传输分为三大类：WebSocket、HTTP 流和 HTTP 长轮询。

SockJS 客户端首先发送 GET /info 以从服务器获取基本信息。之后，它必须决定使用什么传输。如果可能，使用 WebSocket。如果没有，在大多数浏览器中，至少有一个 HTTP 流选项。如果不是，则使用 HTTP（长）轮询。

所有传输请求都具有以下 URL 结构：

```http
https://host:port/myApp/myEndpoint/{server-id}/{session-id}/{transport}
```

在哪里：

- `{server-id}` 用于在集群中路由请求，但不用于其他用途。
- `{session-id}` 关联属于 SockJS 会话的 HTTP 请求。
- `{transport}` 表示传输类型（例如，websocket、xhr-streaming 等）。

WebSocket 传输只需要一个 HTTP 请求来进行 WebSocket 握手。此后的所有消息都在该socket上交换。

HTTP 传输需要更多请求。例如，Ajax/XHR 流依赖于对服务器到客户端消息的一个长时间运行的请求和对客户端到服务器消息的附加 HTTP POST 请求。长轮询类似，不同之处在于它在每次服务器到客户端发送后结束当前请求。

SockJS 添加了最少的消息框架。例如，服务器最初发送字母 o（“打开”帧），消息作为 ["message1","message2"]（JSON 编码数组）发送，如果没有消息，则发送字母 h（“心跳”帧）流 25 秒（默认情况下），并使用字母 c（“关闭”帧）关闭会话。

要了解更多信息，请在浏览器中运行示例并观察 HTTP 请求。 SockJS 客户端允许修复传输列表，因此可以一次查看每个传输一个。 SockJS 客户端还提供了一个调试标志，它可以在浏览器控制台中启用有用的消息。在服务器端，您可以为 org.springframework.web.socket 启用 TRACE 日志记录。有关更多详细信息，请参阅 SockJS 协议叙述测试。

#### 启用 SockJS

您可以通过 Java 配置启用 SockJS，如下例所示：

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(myHandler(), "/myHandler").withSockJS();
    }

    @Bean
    public WebSocketHandler myHandler() {
        return new MyHandler();
    }

}
```

以下示例显示了与前面示例等效的 XML 配置：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:websocket="http://www.springframework.org/schema/websocket"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/websocket
        https://www.springframework.org/schema/websocket/spring-websocket.xsd">

    <websocket:handlers>
        <websocket:mapping path="/myHandler" handler="myHandler"/>
        <websocket:sockjs/>
    </websocket:handlers>

    <bean id="myHandler" class="org.springframework.samples.MyHandler"/>

</beans>
```

前面的示例用于 Spring MVC 应用程序，应包含在 DispatcherServlet 的配置中。但是，Spring 的 WebSocket 和 SockJS 支持不依赖于 Spring MVC。借助 SockJsHttpRequestHandler 集成到其他 HTTP 服务环境中相对简单。

在浏览器端，应用程序可以使用 sockjs-client（版本 1.0.x）。它模拟 W3C WebSocket API 并与服务器通信以选择最佳传输选项，具体取决于它运行的浏览器。请参阅 sockjs-client 页面和浏览器支持的传输类型列表。客户端还提供了几个配置选项 — ，例如，指定要包含哪些传输。

#### IE 8 and 9

Internet Explorer 8 和 9 仍在使用中。它们是拥有 SockJS 的一个关键原因。本节涵盖有关在这些浏览器中运行的重要注意事项。

SockJS 客户端通过使用 Microsoft 的 XDomainRequest 在 IE 8 和 9 中支持 Ajax/XHR 流。这适用于跨域，但不支持发送 cookie。 Cookie 对于 Java 应用程序通常是必不可少的。但是，由于 SockJS 客户端可以与许多服务器类型（不仅仅是 Java 类型）一起使用，因此它需要知道 cookie 是否重要。如果是这样，SockJS 客户端更喜欢使用 Ajax/XHR 进行流式处理。否则，它依赖于基于 iframe 的技术。

来自 SockJS 客户端的第一个 /info 请求是对可能影响客户端传输选择的信息的请求。这些细节之一是服务器应用程序是否依赖 cookie（例如，出于身份验证目的或使用粘性会话进行集群）。 Spring 的 SockJS 支持包括一个名为 sessionCookieNeeded 的属性。它默认启用，因为大多数 Java 应用程序依赖于 JSESSIONID cookie。如果您的应用程序不需要它，您可以关闭此选项，然后 SockJS 客户端应该在 IE 8 和 9 中选择 xdr-streaming。

如果您确实使用基于 iframe 的传输，请记住，可以通过将 HTTP 响应标头 X-Frame-Options 设置为 DENY、SAMEORIGIN 或 ALLOW-FROM <origin 来指示浏览器阻止在给定页面上使用 IFrame >.这用于防止点击劫持。

Spring Security 3.2+ 支持在每个响应上设置 X-Frame-Options。默认情况下，Spring Security Java 配置将其设置为 DENY。在 3.2 中，Spring Security XML 命名空间默认情况下不会设置该标头，但可以配置为这样做。将来，它可能会默认设置。

有关如何配置 X-Frame-Options 标头设置的详细信息，请参阅 Spring Security 文档的默认安全标头。您还可以查看 SEC-2501 以了解更多背景信息。

如果您的应用程序添加了 X-Frame-Options 响应标头（它应该如此！）并依赖于基于 iframe 的传输，则您需要将标头值设置为 SAMEORIGIN 或 ALLOW-FROM <origin>。 Spring SockJS 支持还需要知道 SockJS 客户端的位置，因为它是从 iframe 加载的。默认情况下，iframe 设置为从 CDN 位置下载 SockJS 客户端。将此选项配置为使用与应用程序同源的 URL 是个好主意。

以下示例显示了如何在 Java 配置中执行此操作：

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/portfolio").withSockJS()
                .setClientLibraryUrl("http://localhost:8080/myapp/js/sockjs-client.js");
    }

    // ...

}
```

XML 命名空间通过 <websocket:sockjs> 元素提供了类似的选项。

在初始开发期间，请启用 SockJS 客户端开发模式，以防止浏览器缓存原本会被缓存的 SockJS 请求（如 iframe）。有关如何启用它的详细信息，请参阅 SockJS 客户端页面。

#### Heartbeats

SockJS 协议要求服务器发送心跳消息以防止代理得出连接挂起的结论。 Spring SockJS 配置有一个名为 heartbeatTime 的属性，您可以使用它来自定义频率。默认情况下，心跳在 25 秒后发送，假设该连接上没有发送其他消息。这个 25 秒的值符合以下 IETF 对公共 Internet 应用程序的建议。

在 WebSocket 和 SockJS 上使用 STOMP 时，如果 STOMP 客户端和服务器协商要交换的心跳，则禁用 SockJS 心跳。

Spring SockJS 支持还允许您配置 TaskScheduler 以安排心跳任务。任务调度程序由线程池支持，默认设置基于可用处理器的数量。您应该考虑根据您的特定需求自定义设置。

#### 客户端断开连接

HTTP 流和 HTTP 长轮询 SockJS 传输要求连接比平时保持打开状态的时间更长。

在 Servlet 容器中，这是通过 Servlet 3 异步支持完成的，该支持允许退出 Servlet 容器线程、处理请求并继续写入来自另一个线程的响应。

一个特定的问题是 Servlet API 不为已消失的客户端提供通知。参见 eclipse-ee4j/servlet-api#44。但是，Servlet 容器在后续尝试写入响应时引发异常。由于 Spring 的 SockJS 服务支持服务器发送的心跳（默认情况下每 25 秒），这意味着通常会在该时间段内检测到客户端断开连接（或更早，如果消息发送更频繁）。

因此，由于客户端断开连接，可能会发生网络 I/O 故障，这可能会用不必要的堆栈跟踪填充日志。 Spring 尽最大努力识别代表客户端断开连接（特定于每个服务器）的此类网络故障，并通过使用专用日志类别 DISCONNECTED_CLIENT_LOG_CATEGORY（在 AbstractSockJsSession 中定义）记录最少的消息。如果您需要查看堆栈跟踪，可以将该日志类别设置为 TRACE。

#### SockJS 和 CORS

如果您允许跨域请求（请参阅 Allowed Origins），则 SockJS 协议使用 CORS 在 XHR 流和轮询传输中提供跨域支持。因此，除非检测到响应中存在 CORS 标头，否则会自动添加 CORS 标头。因此，如果应用程序已经配置为提供 CORS 支持（例如，通过 Servlet 过滤器），则 Spring 的 SockJsService 会跳过这部分。

也可以通过设置 Spring 的 SockJsService 中的 suppressCors 属性来禁用这些 CORS 标头的添加。

SockJS 需要以下header和值：

- `Access-Control-Allow-Origin`: 从 Origin 请求标头的值初始化。
- `Access-Control-Allow-Credentials`: 始终设置为真。
- `Access-Control-Request-Headers`: 从等效请求标头中的值初始化。
- `Access-Control-Allow-Methods`: 传输支持的 HTTP 方法（请参阅 TransportType 枚举）。
- `Access-Control-Max-Age`: 设置为 31536000（1 年）。

有关确切的实现，请参阅 AbstractSockJsService 中的 addCorsHeaders 和源代码中的 TransportType 枚举。

或者，如果 CORS 配置允许，请考虑使用 SockJS 端点前缀排除 URL，从而让 Spring 的 SockJsService 处理它。

#### SockJsClient

Spring 提供了一个 SockJS Java 客户端，无需使用浏览器即可连接到远程 SockJS 端点。当需要在公共网络上的两个服务器之间进行双向通信时（即，网络代理可以阻止使用 WebSocket 协议），这尤其有用。 SockJS Java 客户端对于测试目的也非常有用（例如，模拟大量并发用户）。

SockJS Java 客户端支持 websocket、xhr-streaming 和 xhr-polling 传输。其余的仅在浏览器中使用才有意义。

您可以使用以下命令配置 WebSocketTransport：

- `StandardWebSocketClient` 在 JSR-356 运行时中。
- `JettyWebSocketClient` 通过使用 Jetty 9+ 原生 WebSocket API。
- Spring 的 WebSocketClient 的任何实现。

根据定义，XhrTransport 支持 xhr-streaming 和 xhr-polling，因为从客户端的角度来看，除了用于连接到服务器的 URL 之外没有其他区别。目前有两种实现方式：

- `RestTemplateXhrTransport` RestTemplateXhrTransport 使用 Spring 的 RestTemplate 来处理 HTTP 请求。
- `JettyXhrTransport` 使用 Jetty 的 HttpClient 进行 HTTP 请求。

以下示例显示了如何创建 SockJS 客户端并连接到 SockJS 端点：

```java
List<Transport> transports = new ArrayList<>(2);
transports.add(new WebSocketTransport(new StandardWebSocketClient()));
transports.add(new RestTemplateXhrTransport());

SockJsClient sockJsClient = new SockJsClient(transports);
sockJsClient.doHandshake(new MyWebSocketHandler(), "ws://example.com:8080/sockjs");
```

SockJS 对消息使用 JSON 格式的数组。默认情况下，使用 Jackson 2 并且需要在类路径上。或者，您可以配置 SockJsMessageCodec 的自定义实现并在 SockJsClient 上进行配置。

要使用 SockJsClient 模拟大量并发用户，您需要配置底层 HTTP 客户端（用于 XHR 传输）以允许足够数量的连接和线程。以下示例显示了如何使用 Jetty 执行此操作：

```java
HttpClient jettyHttpClient = new HttpClient();
jettyHttpClient.setMaxConnectionsPerDestination(1000);
jettyHttpClient.setExecutor(new QueuedThreadPool(1000));
```

以下示例显示了您还应该考虑自定义的服务器端 SockJS 相关属性（有关详细信息，请参阅 javadoc）：

```java
@Configuration
public class WebSocketConfig extends WebSocketMessageBrokerConfigurationSupport {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/sockjs").withSockJS()
            .setStreamBytesLimit(512 * 1024) 
            .setHttpMessageCacheSize(1000) 
            .setDisconnectDelay(30 * 1000); 
    }

    // ...
}
```

- 将 streamBytesLimit 属性设置为 512KB（默认为 128KB — 128 * 1024）。
- 将 httpMessageCacheSize 属性设置为 1,000（默认值为 100）。
- 将 disconnectDelay 属性设置为 30 属性秒（默认为 5 秒 — 5 * 1000）。

### STOMP

WebSocket 协议定义了两种类型的消息（文本和二进制），但它们的内容是未定义的。该协议定义了客户端和服务器协商一个子协议（即更高级别的消息协议）的机制，用于在 WebSocket 之上定义每个可以发送的消息类型，格式是什么，内容是什么。每条消息，等等。子协议的使用是可选的，但无论哪种方式，客户端和服务器都需要就定义消息内容的某些协议达成一致。

#### 概述

STOMP（面向简单文本的消息传递协议）最初是为脚本语言（例如 Ruby、Python 和 Perl）创建的，用于连接到企业消息代理。它旨在解决常用消息传递模式的最小子集。 STOMP 可用于任何可靠的双向流网络协议，例如 TCP 和 WebSocket。尽管 STOMP 是面向文本的协议，但消息有效负载可以是文本或二进制的。

STOMP 是一种基于帧的协议，其帧以 HTTP 为模型。以下清单显示了 STOMP 帧的结构：

```stomp
COMMAND
header1:value1
header2:value2

Body^@
```

客户端可以使用 SEND 或 SUBSCRIBE 命令发送或订阅消息，以及描述消息内容和接收者的目标头。这启用了一个简单的发布-订阅机制，您可以使用该机制通过代理将消息发送到其他连接的客户端，或将消息发送到服务器以请求执行某些工作。

当您使用 Spring 的 STOMP 支持时，Spring WebSocket 应用程序充当客户端的 STOMP 代理。消息被路由到@Controller 消息处理方法或一个简单的内存代理，该代理跟踪订阅并向订阅用户广播消息。您还可以将 Spring 配置为与专用的 STOMP 代理（例如 RabbitMQ、ActiveMQ 等）一起工作，以进行实际的消息广播。在这种情况下，Spring 维护与代理的 TCP 连接，将消息中继到它，并将消息从它向下传递到连接的 WebSocket 客户端。因此，Spring Web 应用程序可以依靠统一的基于 HTTP 的安全性、通用验证和熟悉的编程模型来处理消息。

以下示例显示订阅接收股票报价的客户端，服务器可能会定期发出该报价（例如，通过计划任务通过 SimpMessagingTemplate 向代理发送消息）：

```
SUBSCRIBE
id:sub-1
destination:/topic/price.stock.*

^@
```

以下示例显示了发送交易请求的客户端，服务器可以通过 @MessageMapping 方法处理该请求：

```
SEND
destination:/queue/trade
content-type:application/json
content-length:44

{"action":"BUY","ticker":"MMM","shares",44}^@
```

执行后，服务器可以向客户端广播交易确认消息和详细信息。

在 STOMP 规范中，目的地的含义故意不透明。它可以是任何字符串，完全由 STOMP 服务器来定义它们支持的目的地的语义和语法。然而，目的地是类似路径的字符串是很常见的，其中 /topic/.. 意味着发布-订阅（一对多）和 /queue/ 意味着点对点（一对一）消息交流。

STOMP 服务器可以使用 MESSAGE 命令向所有订阅者广播消息。以下示例显示服务器向订阅的客户端发送股票报价：

```
MESSAGE
message-id:nxahklf6-1
subscription:sub-1
destination:/topic/price.stock.MMM

{"ticker":"MMM","price":129.45}^@
```

服务器不能发送未经请求的消息。来自服务器的所有消息都必须响应特定的客户端订阅，并且服务器消息的 subscription-id 标头必须与客户端订阅的 id 标头匹配。

前面的概述旨在提供对 STOMP 协议最基本的理解。我们建议全面审查协议规范。

#### 好处

与使用原始 WebSockets 相比，使用 STOMP 作为子协议可以让 Spring Framework 和 Spring Security 提供更丰富的编程模型。对于 HTTP 与原始 TCP 以及它如何让 Spring MVC 和其他 Web 框架提供丰富的功能，可以提出同样的观点。以下是福利清单：

- 无需发明自定义消息传递协议和消息格式。
- 可以使用 STOMP 客户端，包括 Spring 框架中的 Java 客户端。
- 您可以（可选）使用消息代理（例如 RabbitMQ、ActiveMQ 等）来管理订阅和广播消息。
- 应用程序逻辑可以组织在任意数量的 @Controller 实例中，并且可以根据 STOMP 目标标头将消息路由到它们，而不是使用单个 WebSocketHandler 处理给定连接的原始 WebSocket 消息。
- 您可以使用 Spring Security 根据 STOMP 目标和消息类型来保护消息。

#### 启用 STOMP

STOMP over WebSocket 支持在 spring-messaging 和 spring-websocket 模块中可用。一旦有了这些依赖项，就可以通过 WebSocket 和 SockJS Fallback 公开 STOMP 端点，如下例所示：

```java
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/portfolio").withSockJS();  
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.setApplicationDestinationPrefixes("/app"); 
        config.enableSimpleBroker("/topic", "/queue"); 
    }
}
```

- /portfolio 是 WebSocket（或 SockJS）到的端点的 HTTP URL 客户端需要连接以进行 WebSocket 握手。
- 目标标头以 /app 开头的 STOMP 消息被路由到 @Controller 类中的 @MessageMapping 方法。
- 使用内置的消息代理进行订阅和广播以及 将目标标头以 /topic ` 或 `/queue 开头的消息路由到代理。

以下示例显示了与前面示例等效的 XML 配置：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:websocket="http://www.springframework.org/schema/websocket"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/websocket
        https://www.springframework.org/schema/websocket/spring-websocket.xsd">

    <websocket:message-broker application-destination-prefix="/app">
        <websocket:stomp-endpoint path="/portfolio">
            <websocket:sockjs/>
        </websocket:stomp-endpoint>
        <websocket:simple-broker prefix="/topic, /queue"/>
    </websocket:message-broker>

</beans>
```

对于内置的简单代理，/topic 和 /queue 前缀没有任何特殊含义。它们只是一种区分发布-订阅与点对点消息传递（即许多订阅者与一个消费者）的约定。当您使用外部代理时，请查看代理的 STOMP 页面以了解其支持的 STOMP 目的地和前缀类型。

要从浏览器连接，对于 SockJS，您可以使用 sockjs-client。对于 STOMP，许多应用程序使用了 jmesnil/stomp-websocket 库（也称为 stomp.js），该库功能完备，已在生产中使用多年但不再维护。目前，JSteunou/webstomp-client 是该库最积极维护和发展的继承者。以下示例代码基于它：

```javascript
var socket = new SockJS("/spring-websocket-portfolio/portfolio");
var stompClient = webstomp.over(socket);

stompClient.connect({}, function(frame) {
}
```

或者，如果您通过 WebSocket（不带 SockJS）连接，则可以使用以下代码：

```javascript
var socket = new WebSocket("/spring-websocket-portfolio/portfolio");
var stompClient = Stomp.over(socket);

stompClient.connect({}, function(frame) {
}
```

请注意，前面示例中的 stompClient 不需要指定登录和密码标头。即使这样做了，它们也会在服务器端被忽略（或者更确切地说，被覆盖）。有关身份验证的更多信息，请参阅连接到代理和身份验证。

#### WebSocket服务

要配置底层 WebSocket 服务器，服务器配置中的信息适用。但是，对于 Jetty，您需要通过 StompEndpointRegistry 设置 HandshakeHandler 和 WebSocketPolicy：

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/portfolio").setHandshakeHandler(handshakeHandler());
    }

    @Bean
    public DefaultHandshakeHandler handshakeHandler() {

        WebSocketPolicy policy = new WebSocketPolicy(WebSocketBehavior.SERVER);
        policy.setInputBufferSize(8192);
        policy.setIdleTimeout(600000);

        return new DefaultHandshakeHandler(
                new JettyRequestUpgradeStrategy(new WebSocketServerFactory(policy)));
    }
}
```

#### 消息流

一旦暴露了 STOMP 端点，Spring 应用程序就成为连接客户端的 STOMP 代理。本节介绍服务器端的消息流向。

spring-messaging 模块包含对源自 Spring Integration 的消息传递应用程序的基础支持，后来被提取并合并到 Spring 框架中，以便在许多 Spring 项目和应用程序场景中更广泛地使用。以下列表简要描述了一些可用的消息传递抽象：

- [Message](https://docs.spring.io/spring-framework/docs/5.3.12/javadoc-api/org/springframework/messaging/Message.html): 消息的简单表示，包括标头和有效负载。
- [MessageHandler](https://docs.spring.io/spring-framework/docs/5.3.12/javadoc-api/org/springframework/messaging/MessageHandler.html): 处理消息的合同。
- [MessageChannel](https://docs.spring.io/spring-framework/docs/5.3.12/javadoc-api/org/springframework/messaging/MessageChannel.html): 用于发送消息的合同，可以在生产者和消费者之间实现松散耦合。
- [SubscribableChannel](https://docs.spring.io/spring-framework/docs/5.3.12/javadoc-api/org/springframework/messaging/SubscribableChannel.html): 带有 MessageHandler 订阅者的 MessageChannel。
- [ExecutorSubscribableChannel](https://docs.spring.io/spring-framework/docs/5.3.12/javadoc-api/org/springframework/messaging/support/ExecutorSubscribableChannel.html): 使用 Executor 传递消息的 SubscribableChannel。

Java 配置（即@EnableWebSocketMessageBroker）和XML 命名空间配置（即<websocket:message-broker>）都使用前面的组件来组装消息工作流。下图显示了启用简单内置消息代理时使用的组件：

![message flow simple broker](https://docs.spring.io/spring-framework/docs/current/reference/html/images/message-flow-simple-broker.png)



上图显示了三个消息通道：

- `clientInboundChannel`: 用于传递从 WebSocket 客户端接收的消息。
- `clientOutboundChannel`: 用于向 WebSocket 客户端发送服务器消息。
- `brokerChannel`: 用于从服务器端应用程序代码中向消息代理发送消息。

下图显示了在配置外部代理（例如 RabbitMQ）以管理订阅和广播消息时使用的组件：

![message flow broker relay](https://docs.spring.io/spring-framework/docs/current/reference/html/images/message-flow-broker-relay.png)

前面两个图之间的主要区别是使用“代理中继”通过 TCP 将消息向上传递到外部 STOMP 代理，以及将消息从代理向下传递到订阅的客户端。

当从 WebSocket 连接接收到消息时，它们被解码为 STOMP 帧，转换为 Spring Message 表示，并发送到 clientInboundChannel 进行进一步处理。例如，目标标头以 /app 开头的 STOMP 消息可以路由到带注释的控制器中的 @MessageMapping 方法，而 /topic 和 /queue 消息可以直接路由到消息代理。

处理来自客户端的 STOMP 消息的带注释的 @Controller 可以通过 brokerChannel 向消息代理发送消息，并且代理通过 clientOutboundChannel 将消息广播给匹配的订阅者。同一个控制器也可以响应HTTP请求做同样的事情，所以客户端可以执行HTTP POST，然后@PostMapping方法可以向消息代理发送消息广播给订阅的客户端。

我们可以通过一个简单的例子来追踪流程。考虑以下示例，它设置了一个服务器：

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/portfolio");
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.setApplicationDestinationPrefixes("/app");
        registry.enableSimpleBroker("/topic");
    }
}

@Controller
public class GreetingController {

    @MessageMapping("/greeting")
    public String handle(String greeting) {
        return "[" + getTimestamp() + ": " + greeting;
    }
}
```

前面的示例支持以下流程：

1. 客户端连接到 http://localhost:8080/portfolio，一旦建立了 WebSocket 连接，STOMP 帧就开始在其上流动。
2. 客户端发送一个带有 /topic/greeting 目标头的 SUBSCRIBE 帧。收到并解码后，消息将发送到 clientInboundChannel，然后路由到存储客户端订阅的消息代理。
3. 客户端向 /app/greeting 发送 SEND 帧。 /app 前缀有助于将其路由到带注释的控制器。去掉 /app 前缀后，目的地的剩余 /greeting 部分映射到 GreetingController 中的 @MessageMapping 方法。
4. 从 GreetingController 返回的值根据返回值和 /topic/greeting 的默认目标标头（从 /app 替换为 /topic 的输入目标派生）转换为带有有效负载的 Spring 消息。结果消息被发送到 brokerChannel 并由消息代理处理。
5. 消息代理找到所有匹配的订阅者并通过 clientOutboundChannel 向每个订阅者发送一个 MESSAGE 帧，从那里消息被编码为 STOMP 帧并通过 WebSocket 连接发送。

#### 带注释的Controllers

应用程序可以使用带注释的 @Controller 类来处理来自客户端的消息。此类类可以声明 @MessageMapping、@SubscribeMapping 和 @ExceptionHandler 方法，如以下主题中所述：

- [`@MessageMapping`](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#websocket-stomp-message-mapping)
- [`@SubscribeMapping`](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#websocket-stomp-subscribe-mapping)
- [`@MessageExceptionHandler`](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#websocket-stomp-exception-handler)

##### `@MessageMapping`

您可以使用 @MessageMapping 注释根据目的地路由消息的方法。它在方法级别和类型级别都受支持。在类型级别，@MessageMapping 用于表示控制器中所有方法之间的共享映射。

默认情况下，映射值是 Ant 样式的路径模式（例如 /thing*、/thing/**），包括对模板变量的支持（例如 /thing/{id}）。可以通过@DestinationVariable 方法参数引用这些值。应用程序还可以切换到以点分隔的目标约定进行映射，如点作为分隔符中所述。

支持的方法参数

下表描述了方法参数：

| 方法参数                                                     | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `Message`                                                    | 用于访问完整的消息。                                         |
| `MessageHeaders`                                             | 用于访问消息中的标题。                                       |
| `MessageHeaderAccessor`, `SimpMessageHeaderAccessor`, and `StompHeaderAccessor` | 用于通过类型化访问器方法访问标头。                           |
| `@Payload`                                                   | 用于访问消息的有效负载，由配置的 MessageConverter 转换（例如，从 JSON）。 不需要此注释的存在，因为默认情况下，如果没有其他参数匹配，则假定它存在。 您可以使用 @javax.validation.Valid 或 Spring 的 @Validated 注释负载参数，以自动验证负载参数。 |
| `@Header`                                                    | 用于访问特定标头值 — 以及使用 org.springframework.core.convert.converter.Converter 进行类型转换（如有必要）。 |
| `@Headers`                                                   | 用于访问消息中的所有标头。此参数必须可分配给 java.util.Map。 |
| `@DestinationVariable`                                       | 用于访问从消息目标中提取的模板变量。根据需要将值转换为声明的方法参数类型。 |
| `java.security.Principal`                                    | 反映在 WebSocket HTTP 握手时登录的用户。                     |

返回值

默认情况下，@MessageMapping 方法的返回值通过匹配的 MessageConverter 序列化为有效负载，并作为消息发送到 brokerChannel，从那里广播给订阅者。出站消息的目的地与入站消息的目的地相同，但以 /topic 为前缀。

您可以使用 @SendTo 和 @SendToUser 注释来自定义输出消息的目的地。 @SendTo 用于自定义目标目的地或指定多个目的地。 @SendToUser 用于将输出消息定向到与输入消息关联的用户。

您可以在同一方法上同时使用 @SendTo 和 @SendToUser，并且在类级别都支持两者，在这种情况下，它们充当类中方法的默认值。但是，请记住，任何方法级别的 @SendTo 或 @SendToUser 注释都会覆盖类级别的任何此类注释。

消息可以异步处理，@MessageMapping 方法可以返回 ListenableFuture、CompletableFuture 或 CompletionStage。

请注意，@SendTo 和@SendToUser 只是一种便利，相当于使用 SimpMessagingTemplate 发送消息。如有必要，对于更高级的场景，@MessageMapping 方法可以直接使用 SimpMessagingTemplate。这可以代替返回值来完成，或者除了返回值之外。请参阅发送消息。

##### `@SubscribeMapping`

@SubscribeMapping 与@MessageMapping 类似，但将映射范围缩小到仅订阅消息。它支持与@MessageMapping 相同的方法参数。但是对于返回值，默认情况下，消息直接发送到客户端（通过 clientOutboundChannel，响应订阅）而不是发送到代理（通过 brokerChannel，作为匹配订阅的广播）。添加 @SendTo 或 @SendToUser 会覆盖此行为并改为发送到代理。

这什么时候有用？假设代理映射到 /topic 和 /queue，而应用程序控制器映射到 /app。在此设置中，代理存储所有用于重复广播的 /topic 和 /queue 订阅，应用程序无需参与。客户端还可以订阅某个 /app 目的地，并且控制器可以返回一个值来响应该订阅，而无需代理再次存储或使用订阅（实际上是一次请求-回复交换）。一个用例是在启动时使用初始数据填充 UI。

这什么时候没用？不要尝试将代理和控制器映射到相同的目标前缀，除非您出于某种原因希望两者独立处理消息，包括订阅。入站消息是并行处理的。不能保证是代理还是控制器首先处理给定的消息。如果目标是在订阅存储并准备好广播时收到通知，如果服务器支持，客户端应该要求收据（简单代理不支持）。例如，使用 Java STOMP 客户端，您可以执行以下操作来添加收据：

```java
@Autowired
private TaskScheduler messageBrokerTaskScheduler;

// During initialization..
stompClient.setTaskScheduler(this.messageBrokerTaskScheduler);

// When subscribing..
StompHeaders headers = new StompHeaders();
headers.setDestination("/topic/...");
headers.setReceipt("r1");
FrameHandler handler = ...;
stompSession.subscribe(headers, handler).addReceiptTask(() -> {
    // Subscription ready...
});
```

服务器端选项是在 brokerChannel 上注册一个 ExecutorChannelInterceptor 并实现 afterMessageHandled 方法，该方法在处理消息（包括订阅）后调用。

##### `@MessageExceptionHandler`

应用程序可以使用@MessageExceptionHandler 方法来处理来自@MessageMapping 方法的异常。如果您想访问异常实例，您可以在注释本身或通过方法参数声明异常。以下示例通过方法参数声明异常：

```java
@Controller
public class MyController {

    // ...

    @MessageExceptionHandler
    public ApplicationError handleException(MyException exception) {
        // ...
        return appError;
    }
}
```

@MessageExceptionHandler 方法支持灵活的方法签名，并支持与@MessageMapping 方法相同的方法参数类型和返回值。

通常，@MessageExceptionHandler 方法应用于声明它们的 @Controller 类（或类层次结构）中。如果您希望这些方法在全局范围内（跨控制器）应用，您可以在标有 @ControllerAdvice 的类中声明它们。这与 Spring MVC 中可用的类似支持相当。

#### 发送消息

如果您想从应用程序的任何部分向连接的客户端发送消息怎么办？任何应用程序组件都可以向 brokerChannel 发送消息。最简单的方法是注入一个 SimpMessagingTemplate 并使用它来发送消息。通常，您会按类型注入它，如以下示例所示：

```java
@Controller
public class GreetingController {

    private SimpMessagingTemplate template;

    @Autowired
    public GreetingController(SimpMessagingTemplate template) {
        this.template = template;
    }

    @RequestMapping(path="/greetings", method=POST)
    public void greet(String greeting) {
        String text = "[" + getTimestamp() + "]:" + greeting;
        this.template.convertAndSend("/topic/greetings", text);
    }

}
```

但是，如果存在另一个相同类型的 bean，您也可以通过其名称 (brokerMessagingTemplate) 对其进行限定。

#### Simple Broker

内置的简单消息代理处理来自客户端的订阅请求，将它们存储在内存中，并将消息广播到具有匹配目的地的连接客户端。代理支持类似路径的目的地，包括订阅 Ant 风格的目的地模式。

应用程序还可以使用点分隔（而不是斜线分隔）目标。将点视为分隔符。

如果配置了任务调度程序，则简单代理支持 STOMP 心跳。为此，您可以声明自己的调度程序或使用自动声明并在内部使用的调度程序。以下示例显示了如何声明您自己的调度程序：

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    private TaskScheduler messageBrokerTaskScheduler;

    @Autowired
    public void setMessageBrokerTaskScheduler(TaskScheduler taskScheduler) {
        this.messageBrokerTaskScheduler = taskScheduler;
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {

        registry.enableSimpleBroker("/queue/", "/topic/")
                .setHeartbeatValue(new long[] {10000, 20000})
                .setTaskScheduler(this.messageBrokerTaskScheduler);

        // ...
    }
}
```

#### External Broker

simple broker 非常适合入门，但仅支持 STOMP 命令的一个子集（它不支持 acks、receipts 和一些其他功能），依赖于简单的消息发送循环，并且不适合集群。作为替代方案，您可以升级您的应用程序以使用功能齐全的消息代理。

请参阅您选择的消息代理（例如 RabbitMQ、ActiveMQ 等）的 STOMP 文档，安装代理，并在启用 STOMP 支持的情况下运行它。然后你可以在 Spring 配置中启用 STOMP 代理中继（而不是简单的代理）。

以下示例配置启用全功能代理：

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/portfolio").withSockJS();
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableStompBrokerRelay("/topic", "/queue");
        registry.setApplicationDestinationPrefixes("/app");
    }

}

```

以下示例显示了与前面示例等效的 XML 配置：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:websocket="http://www.springframework.org/schema/websocket"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/websocket
        https://www.springframework.org/schema/websocket/spring-websocket.xsd">

    <websocket:message-broker application-destination-prefix="/app">
        <websocket:stomp-endpoint path="/portfolio" />
            <websocket:sockjs/>
        </websocket:stomp-endpoint>
        <websocket:stomp-broker-relay prefix="/topic,/queue" />
    </websocket:message-broker>

</beans>
```

前面配置中的 STOMP 代理中继是一个 Spring MessageHandler，它通过将消息转发到外部消息代理来处理消息。为此，它会与代理建立 TCP 连接，将所有消息转发给它，然后通过客户端的 WebSocket 会话将从代理收到的所有消息转发给客户端。本质上，它充当双向转发消息的“中继”。

将 io.projectreactor.netty:reactor-netty 和 io.netty:netty-all 依赖项添加到您的项目中以进行 TCP 连接管理。

此外，应用程序组件（例如 HTTP 请求处理方法、业务服务等）还可以将消息发送到代理中继，如发送消息中所述，以向订阅的 WebSocket 客户端广播消息。

实际上，代理中继实现了强大且可扩展的消息广播。

