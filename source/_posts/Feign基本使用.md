---
title: Feign基本使用
date: 2021-11-09 08:12:43
tags: Feign
categories: Feign
---

# Feign基本使用

## 1、功能概述

![img](https://camo.githubusercontent.com/f1bd8b9bfe3c049484b0776b42668bb76a57872fe0f01402e5ef73d29b811e50/687474703a2f2f7777772e706c616e74756d6c2e636f6d2f706c616e74756d6c2f70726f78793f63616368653d6e6f267372633d68747470733a2f2f7261772e67697468756275736572636f6e74656e742e636f6d2f4f70656e466569676e2f666569676e2f6d61737465722f7372632f646f63732f6f766572766965772d6d696e646d61702e69756d6c)

## 2、自定义错误处理

Feign（默认配置）对于所有错误情况只抛出FeignException异常，但是如果你想处理一个特殊的异常，可以通过在Feign.builder.errorDecoder()实现feign.codec.ErrorDecoder接口来达到目的。

例子：

自定义错误处理

```java
public class StashErrorDecoder implements ErrorDecoder {

    @Override
    public Exception decode(String methodKey, Response response) {
        if (response.status() >= 400 && response.status() <= 499) {
            return new StashClientException(
                    response.status(),
                    response.reason()
            );
        }
        if (response.status() >= 500 && response.status() <= 599) {
            return new StashServerException(
                    response.status(),
                    response.reason()
            );
        }
        return errorStatus(methodKey, response);
    }
}
```

应用错误处理

```java
return Feign.builder()
                .errorDecoder(new StashErrorDecoder())
                .target(StashApi.class, url);
```

## 3、基本用法

例子：

```java
interface GitHub {
  @RequestLine("GET /repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@Param("owner") String owner, @Param("repo") String repo);

  @RequestLine("POST /repos/{owner}/{repo}/issues")
  void createIssue(Issue issue, @Param("owner") String owner, @Param("repo") String repo);

}

public static class Contributor {
  String login;
  int contributions;
}

public static class Issue {
  String title;
  String body;
  List<String> assignees;
  int milestone;
  List<String> labels;
}

public class MyApp {
  public static void main(String... args) {
    GitHub github = Feign.builder()
                         .decoder(new GsonDecoder())
                         .target(GitHub.class, "https://api.github.com");

    // Fetch and print a list of the contributors to this library.
    List<Contributor> contributors = github.contributors("OpenFeign", "feign");
    for (Contributor contributor : contributors) {
      System.out.println(contributor.login + " (" + contributor.contributions + ")");
    }
  }
}
```

### 3.1、接口注解

Feign的Contract定义的注解规定了底层客户端和接口之间如何工作。

Feign的默认Contract定义了以下注解：

| 注解         | 接口对象     | 用法                                                         |
| ------------ | ------------ | ------------------------------------------------------------ |
| @RequestLine | Method       | 为请求定义 HttpMethod 和 UriTemplate。表达式，用大括号 {expression} 包裹的值使用其相应的 @Param 注释参数解析。 |
| @Param       | Parameter    | 定义一个模板变量，其值将用于解析相应的模板表达式，按名称提供作为注释值。 如果缺少值，它将尝试从字节码方法参数名称中获取名称（如果代码是使用 -parameters 标志编译的）。 |
| @Headers     | Method, Type | 定义一个 HeaderTemplate； UriTemplate 的变体。使用@Param 注释值来解析相应的表达式。在 Type 上使用时，模板将应用于每个请求。当在method上使用时，模板将仅适用于带注释的方法。 |
| @QueryMap    | Parameter    | 定义名称-值对的 Map 或 POJO，以扩展为查询字符串。            |
| @HeaderMap   | Parameter    | 定义一个名称-值对的 Map，以扩展到 Http Headers。             |
| @Body        | Method       | 定义一个模板，类似于 UriTemplate 和 HeaderTemplate，它使用@Param 注释值来解析相应的表达式。 |

### 3.2、模板和表达式

Feign 表达式表示由 URI Template - RFC 6570 定义的简单字符串表达式（Level 1）。表达式使用其相应的 Param 注释方法参数进行扩展。

例子：

```java
public interface GitHub {

  @RequestLine("GET /repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@Param("owner") String owner, @Param("repo") String repository);

  class Contributor {
    String login;
    int contributions;
  }
}

public class MyApp {
  public static void main(String[] args) {
    GitHub github = Feign.builder()
                         .decoder(new GsonDecoder())
                         .target(GitHub.class, "https://api.github.com");

    /* The owner and repository parameters will be used to expand the owner and repo expressions
     * defined in the RequestLine.
     *
     * the resulting uri will be https://api.github.com/repos/OpenFeign/feign/contributors
     */
    github.contributors("OpenFeign", "feign");
  }
}
```

表达式必须用大括号 {} 括起来，并且可以包含正则表达式模式，用冒号 : 分隔以限制解析的值。示例owner必须按字母顺序排列。 {owner：[a-zA-Z]*}

### 3.3、请求参数扩展

RequestLine 和 QueryMap 模板遵循 URI Template - RFC 6570 Level 1 模板规范，其中指定了以下内容：

- 未解析的表达式被省略。
- 所有文字和变量值都是 pct 编码的，如果尚未编码或标记为通过 @Param 注释编码。

### 3.4、未定义与空值

未定义的表达式是表达式的值为显式 null 或未提供值的表达式。根据 URI Template - RFC 6570，可以为表达式提供空值。 Feign 解析表达式时，它首先确定该值是否已定义，如果已定义，则查询参数将保留。如果表达式未定义，则删除查询参数。请参阅下面的完整细分。

空字符串

```java
public void test() {
   Map<String, Object> parameters = new LinkedHashMap<>();
   parameters.put("param", "");
   this.demoClient.test(parameters);
}
```

结果

```java
http://localhost:8080/test?param=
```

丢失

```java
public void test() {
   Map<String, Object> parameters = new LinkedHashMap<>();
   this.demoClient.test(parameters);
}
```

结果

```java
http://localhost:8080/test
```

未定义

```java
public void test() {
   Map<String, Object> parameters = new LinkedHashMap<>();
   parameters.put("param", null);
   this.demoClient.test(parameters);
}
```

结果

```java
http://localhost:8080/test
```

### 3.5、自定义扩展

@Param 注释有一个可选的属性扩展器，允许完全控制单个参数的扩展。 expander 属性必须引用一个实现 Expander 接口的类：

```java
public interface Expander {
    String expand(Object value);
}
```

### 3.6、请求标头扩展

Headers 和 HeaderMap 模板遵循与请求参数扩展相同的规则，但有以下变化：

- 未解析的表达式被省略。如果结果是一个空的标头值，则整个标头将被删除。
- 不执行 pct 编码。

关于@Param 参数及其名称的说明：

所有具有相同名称的表达式，无论它们在 @RequestLine、@QueryMap、@BodyTemplate 或 @Headers 上的位置如何，都将解析为相同的值。在以下示例中，contentType 的值将用于解析标头和路径表达式：

```java
public interface ContentService {
  @RequestLine("GET /api/documents/{contentType}")
  @Headers("Accept: {contentType}")
  String getDocumentByType(@Param("contentType") String type);
}
```

### 3.7、请求正文扩展

正文模板遵循与请求参数扩展相同的规则，但有以下更改：

- 未解析的表达式被省略。
- 扩展的值在放置到请求正文之前不会通过编码器传递。
- 必须指定 Content-Type 标头。

### 3.8、定制

Feign 有几个方面可以定制。 对于简单的情况，您可以使用 Feign.builder() 来构建带有自定义组件的 API 接口。 对于请求设置，您可以使用 target() 上的 options(Request.Options options) 来设置 connectTimeout、connectTimeoutUnit、readTimeout、readTimeoutUnit、followRedirects。 例如：

```java
interface Bank {
  @RequestLine("POST /account/{id}")
  Account getAccountInfo(@Param("id") String id);
}

public class BankService {
  public static void main(String[] args) {
    Bank bank = Feign.builder()
        .decoder(new AccountDecoder())
        .options(new Request.Options(10, TimeUnit.SECONDS, 60, TimeUnit.SECONDS, true))
        .target(Bank.class, "https://api.examplebank.com");
  }
}
```

### 3.9、多个接口

Feign 可以产生多个 api 接口。这些被定义为 Target<T>（默认 HardCodedTarget<T>），允许在执行之前动态发现和修饰请求。 例如，以下模式可能会使用来自身份服务的当前 url 和身份验证令牌来修饰每个请求。

```java
public class CloudService {
  public static void main(String[] args) {
    CloudDNS cloudDNS = Feign.builder()
      .target(new CloudIdentityTarget<CloudDNS>(user, apiKey));
  }

  class CloudIdentityTarget extends Target<CloudDNS> {
    /* implementation of a Target */
  }
}
```

### 3.10、集成

#### 3.10.1、Gson

Gson 包含一个编码器和解码器，您可以将其与 JSON API 一起使用。 像这样将 GsonEncoder 和/或 GsonDecoder 添加到 Feign.Builder 中：

```java
public class Example {
  public static void main(String[] args) {
    GsonCodec codec = new GsonCodec();
    GitHub github = Feign.builder()
                         .encoder(new GsonEncoder())
                         .decoder(new GsonDecoder())
                         .target(GitHub.class, "https://api.github.com");
  }
}
```

#### 3.10.2、Jackson

Jackson 包含一个编码器和解码器，您可以将其与 JSON API 一起使用。 将 JacksonEncoder 和/或 JacksonDecoder 添加到您的 Feign.Builder 中，如下所示：

```java
public class Example {
  public static void main(String[] args) {
      GitHub github = Feign.builder()
                     .encoder(new JacksonEncoder())
                     .decoder(new JacksonDecoder())
                     .target(GitHub.class, "https://api.github.com");
  }
}
```

#### 3.10.3、Sax

SaxDecoder 允许您以与普通 JVM 和 Android 环境兼容的方式解码 XML。 以下是如何配置 Sax 响应解析的示例：

```java
public class Example {
  public static void main(String[] args) {
      Api api = Feign.builder()
         .decoder(SAXDecoder.builder()
                            .registerContentHandler(UserIdHandler.class)
                            .build())
         .target(Api.class, "https://apihost");
    }
}
```

#### 3.10.4、JAXB

JAXB 包括可与 XML API 一起使用的编码器和解码器。 将 JAXBEncoder 和/或 JAXBDecoder 添加到您的 Feign.Builder 中，如下所示：

```java
public class Example {
  public static void main(String[] args) {
    Api api = Feign.builder()
             .encoder(new JAXBEncoder())
             .decoder(new JAXBDecoder())
             .target(Api.class, "https://apihost");
  }
}
```

#### 3.10.5、JAX-RS

JAXRSContract 覆盖注解处理以使用 JAX-RS 规范提供的标准处理。这是目前针对 1.1 规范的目标。 这是上面重写的示例以使用 JAX-RS：

```java
interface GitHub {
  @GET @Path("/repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@PathParam("owner") String owner, @PathParam("repo") String repo);
}

public class Example {
  public static void main(String[] args) {
    GitHub github = Feign.builder()
                       .contract(new JAXRSContract())
                       .target(GitHub.class, "https://api.github.com");
  }
}
```

#### 3.10.6、OkHttp

OkHttpClient 将 Feign 的 http 请求定向到 OkHttp，从而实现 SPDY 和更好的网络控制。 要将 OkHttp 与 Feign 一起使用，请将 OkHttp 模块添加到您的类路径中。然后，配置 Feign 以使用 OkHttpClient：

```java
public class Example {
  public static void main(String[] args) {
    GitHub github = Feign.builder()
                     .client(new OkHttpClient())
                     .target(GitHub.class, "https://api.github.com");
  }
}
```

#### 3.10.7、Ribbon

RibbonClient 覆盖了 Feign 客户端的 URL 解析，增加了 Ribbon 提供的智能路由和弹性能力。 集成要求您将功能区客户端名称作为 url 的主机部分传递，例如 myAppProd。

```java
public class Example {
  public static void main(String[] args) {
    MyService api = Feign.builder()
          .client(RibbonClient.create())
          .target(MyService.class, "https://myAppProd");
  }
}
```

#### 3.10.8、Java 11 Http2

Http2Client 将 Feign 的 http 请求定向到实现 HTTP/2 的 Java11 New HTTP/2 Client。 要将新 HTTP/2 客户端与 Feign 一起使用，请使用 Java SDK 11。然后，将 Feign 配置为使用 Http2Client：

```java
GitHub github = Feign.builder()
                     .client(new Http2Client())
                     .target(GitHub.class, "https://api.github.com");
```

#### 3.10.9、Hystrix

HystrixFeign 配置了 Hystrix 提供的断路器支持。 要将 Hystrix 与 Feign 一起使用，请将 Hystrix 模块添加到您的类路径中。然后使用 HystrixFeign 构建器：

```java
public class Example {
  public static void main(String[] args) {
    MyService api = HystrixFeign.builder().target(MyService.class, "https://myAppProd");
  }
}
```

#### 3.10.10、SOAP

SOAP 包括可以与 XML API 一起使用的编码器和解码器。 该模块添加了对通过 JAXB 和 SOAPMessage 编码和解码 SOAP Body 对象的支持。它还通过将它们包装到原始 javax.xml.ws.soap.SOAPFaultException 中来提供 SOAPFault 解码功能，因此您只需捕获 SOAPFaultException 即可处理 SOAPFault。 像这样将 SOAPEncoder 和/或 SOAPDecoder 添加到你的 Feign.Builder 中：

```java
public class Example {
  public static void main(String[] args) {
    Api api = Feign.builder()
	     .encoder(new SOAPEncoder(jaxbFactory))
	     .decoder(new SOAPDecoder(jaxbFactory))
	     .errorDecoder(new SOAPErrorDecoder())
	     .target(MyApi.class, "http://api");
  }
}
```

#### 3.10.11、SLF4J

SLF4JModule 允许将 Feign 的日志记录定向到 SLF4J，允许您轻松使用您选择的日志记录后端（Logback、Log4J 等） 要将 SLF4J 与 Feign 一起使用，请将 SLF4J 模块和您选择的 SLF4J 绑定添加到您的类路径中。然后，配置 Feign 以使用 Slf4jLogger：

```java
public class Example {
  public static void main(String[] args) {
    GitHub github = Feign.builder()
                     .logger(new Slf4jLogger())
                     .logLevel(Level.FULL)
                     .target(GitHub.class, "https://api.github.com");
  }
}
```

### 3.11、解码器

Feign.builder() 允许您指定其他配置，例如如何解码响应。 如果接口中的任何方法返回 Response、String、byte[] 或 void 之外的类型，则需要配置非默认解码器。 下面是如何配置 JSON 解码（使用 feign-gson 扩展）：

```java
public class Example {
  public static void main(String[] args) {
    GitHub github = Feign.builder()
                     .decoder(new GsonDecoder())
                     .target(GitHub.class, "https://api.github.com");
  }
}
```

如果您需要在将响应提供给解码器之前对其进行预处理，则可以使用 mapAndDecode 构建器方法。一个示例用例是处理仅提供 jsonp 的 API，您可能需要在将其发送到您选择的 Json 解码器之前解开 jsonp：

```java
public class Example {
  public static void main(String[] args) {
    JsonpApi jsonpApi = Feign.builder()
                         .mapAndDecode((response, type) -> jsopUnwrap(response, type), new GsonDecoder())
                         .target(JsonpApi.class, "https://some-jsonp-api.com");
  }
}
```

### 3.12、编码器

将请求正文发送到服务器的最简单方法是定义一个 POST 方法，该方法有一个 String 或 byte[] 参数，上面没有任何注释。您可能需要添加一个 Content-Type 标头。

```java
interface LoginClient {
  @RequestLine("POST /")
  @Headers("Content-Type: application/json")
  void login(String content);
}

public class Example {
  public static void main(String[] args) {
    client.login("{\"user_name\": \"denominator\", \"password\": \"secret\"}");
  }
}
```

通过配置编码器，您可以发送类型安全的请求正文。这是使用 feign-gson 扩展的示例：

```java
static class Credentials {
  final String user_name;
  final String password;

  Credentials(String user_name, String password) {
    this.user_name = user_name;
    this.password = password;
  }
}

interface LoginClient {
  @RequestLine("POST /")
  void login(Credentials creds);
}

public class Example {
  public static void main(String[] args) {
    LoginClient client = Feign.builder()
                              .encoder(new GsonEncoder())
                              .target(LoginClient.class, "https://foo.com");

    client.login(new Credentials("denominator", "secret"));
  }
}
```

### 3.13、@Body 模板

@Body 注释指示使用@Param 注释的参数扩展的模板。您可能需要添加一个 Content-Type 标头。

```java
interface LoginClient {

  @RequestLine("POST /")
  @Headers("Content-Type: application/xml")
  @Body("<login \"user_name\"=\"{user_name}\" \"password\"=\"{password}\"/>")
  void xml(@Param("user_name") String user, @Param("password") String password);

  @RequestLine("POST /")
  @Headers("Content-Type: application/json")
  // json curly braces must be escaped!
  @Body("%7B\"user_name\": \"{user_name}\", \"password\": \"{password}\"%7D")
  void json(@Param("user_name") String user, @Param("password") String password);
}

public class Example {
  public static void main(String[] args) {
    client.xml("denominator", "secret"); // <login "user_name"="denominator" "password"="secret"/>
    client.json("denominator", "secret"); // {"user_name": "denominator", "password": "secret"}
  }
}
```

### 3.14、请求头

Feign 支持请求的设置标头作为 api 的一部分或作为客户端的一部分，具体取决于用例。

#### 3.14.1、使用 apis 设置标题

在特定接口或调用应始终设置某些标头值的情况下，将标头定义为 api 的一部分是有意义的。 可以使用 @Headers 注释在 api 接口或方法上设置静态标头。

可以使用 @Headers 注释在 api 接口或方法上设置静态标头。

```java
@Headers("Accept: application/json")
interface BaseApi<V> {
  @Headers("Content-Type: application/json")
  @RequestLine("PUT /api/{key}")
  void put(@Param("key") String key, V value);
}
```

方法可以使用@Headers 中的变量扩展为静态标题指定动态内容。

```java
public interface Api {
   @RequestLine("POST /")
   @Headers("X-Ping: {token}")
   void post(@Param("token") String token);
}
```

如果标头字段键和值都是动态的，并且可能键的范围无法提前知道，并且可能因同一 api/客户端中的不同方法调用而异（例如自定义元数据标头字段，例如“x-amz- meta-*" 或 "x-goog-meta-*")，可以使用 HeaderMap 对 Map 参数进行注释，以构造使用地图内容作为其标头参数的查询。

```java
public interface Api {
   @RequestLine("POST /")
   void post(@HeaderMap Map<String, Object> headerMap);
}
```

这些方法将标头条目指定为 api 的一部分，并且在构建 Feign 客户端时不需要任何自定义。

#### 3.14.2、为每个目标设置请求头

要为 Target 上的每个请求方法自定义请求头，可以使用 RequestInterceptor。 RequestInterceptors 可以跨 Target 实例共享，并且应该是线程安全的。 RequestInterceptors 应用于 Target 上的所有请求方法。 如果您需要按方法自定义，则需要自定义 Target，因为 RequestInterceptor 无权访问当前方法元数据。 有关使用 RequestInterceptor 设置标头的示例，请参阅请求拦截器部分。 请求头可以设置为自定义目标的一部分。

```java
static class DynamicAuthTokenTarget<T> implements Target<T> {
    public DynamicAuthTokenTarget(Class<T> clazz,
                                  UrlAndTokenProvider provider,
                                  ThreadLocal<String> requestIdProvider);

    @Override
    public Request apply(RequestTemplate input) {
      TokenIdAndPublicURL urlAndToken = provider.get();
      if (input.url().indexOf("http") != 0) {
        input.insert(0, urlAndToken.publicURL);
      }
      input.header("X-Auth-Token", urlAndToken.tokenId);
      input.header("X-Request-ID", requestIdProvider.get());

      return input.request();
    }
  }

  public class Example {
    public static void main(String[] args) {
      Bank bank = Feign.builder()
              .target(new DynamicAuthTokenTarget(Bank.class, provider, requestIdProvider));
    }
  }
```

这些方法取决于在构建 Feign 客户端时在 Feign 客户端上设置的自定义 RequestInterceptor 或 Target，并且可以用作在每个客户端的基础上在所有 api 调用上设置标头的方法。这对于执行诸如在每个客户端的所有 api 请求的标头中设置身份验证令牌等操作非常有用。当在调用 api 调用的线程上进行 api 调用时运行这些方法，这允许在调用时以特定于上下文的方式动态设置标头——例如，线程本地存储可用于根据调用线程设置不同的标头值，这对于诸如为请求设置线程特定的跟踪标识符之类的事情很有用。

## 4、高级用法

### 4.1、基础Apis

在许多情况下，服务的 api 遵循相同的约定。 Feign 通过单继承接口支持这种模式。 考虑这个例子：

```java
interface BaseAPI {
  @RequestLine("GET /health")
  String health();

  @RequestLine("GET /all")
  List<Entity> all();
}
```

您可以定义和定位特定的 api，继承基本方法。

```java
interface CustomAPI extends BaseAPI {
  @RequestLine("GET /custom")
  String custom();
}
```

在许多情况下，资源表示也是一致的。因此，基本 api 接口支持类型参数。

```java
@Headers("Accept: application/json")
interface BaseApi<V> {

  @RequestLine("GET /api/{key}")
  V get(@Param("key") String key);

  @RequestLine("GET /api")
  List<V> list();

  @Headers("Content-Type: application/json")
  @RequestLine("PUT /api/{key}")
  void put(@Param("key") String key, V value);
}

interface FooApi extends BaseApi<Foo> { }

interface BarApi extends BaseApi<Bar> { }
```

### 4.2、日志记录

您可以通过设置 Logger 来记录进出目标的 http 消息。这是最简单的方法：

```java
public class Example {
  public static void main(String[] args) {
    GitHub github = Feign.builder()
                     .decoder(new GsonDecoder())
                     .logger(new Logger.JavaLogger("GitHub.Logger").appendToFile("logs/http.log"))
                     .logLevel(Logger.Level.FULL)
                     .target(GitHub.class, "https://api.github.com");
  }
}
```

关于 JavaLogger 的注意事项：避免使用默认的 JavaLogger() 构造函数 - 它被标记为已弃用，很快将被删除。

### 4.3、请求拦截器

当您需要更改所有请求时，无论其目标是什么，您都需要配置一个 RequestInterceptor。例如，如果您充当中介，您可能想要传播 X-Forwarded-For 标头。

```java
static class ForwardedForInterceptor implements RequestInterceptor {
  @Override public void apply(RequestTemplate template) {
    template.header("X-Forwarded-For", "origin.host.com");
  }
}

public class Example {
  public static void main(String[] args) {
    Bank bank = Feign.builder()
                 .decoder(accountDecoder)
                 .requestInterceptor(new ForwardedForInterceptor())
                 .target(Bank.class, "https://api.examplebank.com");
  }
}
```

拦截器的另一个常见示例是身份验证，例如使用内置的 BasicAuthRequestInterceptor。

```java
public class Example {
  public static void main(String[] args) {
    Bank bank = Feign.builder()
                 .decoder(accountDecoder)
                 .requestInterceptor(new BasicAuthRequestInterceptor(username, password))
                 .target(Bank.class, "https://api.examplebank.com");
  }
}
```

### 4.4、自定义@Param 扩展

使用 Param 注释的参数基于它们的 toString 展开。通过指定自定义 Param.Expander，用户可以控制此行为，例如格式化日期。

```java
public interface Api {
  @RequestLine("GET /?since={date}") Result list(@Param(value = "date", expander = DateToMillis.class) Date date);
}
```

### 4.5、动态查询参数

可以使用 QueryMap 对 Map 参数进行注释，以构造使用地图内容作为其查询参数的查询。

```java
public interface Api {
  @RequestLine("GET /find")
  V find(@QueryMap Map<String, Object> queryMap);
}
```

这也可用于使用 QueryMapEncoder 从 POJO 对象生成查询参数。

```java
public interface Api {
  @RequestLine("GET /find")
  V find(@QueryMap CustomPojo customPojo);
}
```

以这种方式使用时，在不指定自定义 QueryMapEncoder 的情况下，将使用成员变量名称作为查询参数名称生成查询映射。下面的 POJO 将生成“/find?name={name}&number={number}”的查询参数（不保证包含的查询参数的顺序，和往常一样，如果任何值为空，它将被排除在外）。

```java
public class CustomPojo {
  private final String name;
  private final int number;

  public CustomPojo (String name, int number) {
    this.name = name;
    this.number = number;
  }
}
```

要设置自定义 QueryMapEncoder：

```java
public class Example {
  public static void main(String[] args) {
    MyApi myApi = Feign.builder()
                 .queryMapEncoder(new MyCustomQueryMapEncoder())
                 .target(MyApi.class, "https://api.hostname.com");
  }
}
```

使用@QueryMap 注释对象时，默认编码器使用反射来检查提供的对象字段以将对象值扩展为查询字符串。如果您希望使用 Java Beans API 中定义的 getter 和 setter 方法构建查询字符串，请使用 BeanQueryMapEncoder

```java
public class Example {
  public static void main(String[] args) {
    MyApi myApi = Feign.builder()
                 .queryMapEncoder(new BeanQueryMapEncoder())
                 .target(MyApi.class, "https://api.hostname.com");
  }
}
```

### 4.6、错误处理

如果您需要更多控制处理意外响应，Feign 实例可以通过构建器注册自定义 ErrorDecoder。

```java
public class Example {
  public static void main(String[] args) {
    MyApi myApi = Feign.builder()
                 .errorDecoder(new MyErrorDecoder())
                 .target(MyApi.class, "https://api.hostname.com");
  }
}
```

导致 HTTP 状态不在 2xx 范围内的所有响应都将触发 ErrorDecoder 的 decode 方法，允许您处理响应、将失败包装到自定义异常中或执行任何其他处理。如果您想再次重试请求，请抛出 RetryableException。这将调用注册的重试器。

### 4.7、重试

默认情况下，Feign 会自动重试 IOExceptions，不管 HTTP 方法如何，将它们视为与网络相关的瞬态异常，以及从 ErrorDecoder 抛出的任何 RetryableException。要自定义此行为，请通过构建器注册自定义 Retryer 实例。

```java
public class Example {
  public static void main(String[] args) {
    MyApi myApi = Feign.builder()
                 .retryer(new MyRetryer())
                 .target(MyApi.class, "https://api.hostname.com");
  }
}
```

重试器负责通过从方法 continueOrPropagate(RetryableException e); 返回 true 或 false 来确定是否应该进行重试；将为每个客户端执行创建一个 Retryer 实例，允许您根据需要维护每个请求之间的状态。 如果确定重试不成功，则会抛出最后一个 RetryException。要抛出导致重试失败的原始原因，请使用 exceptionPropagationPolicy() 选项构建您的 Feign 客户端。

### 4.8、指标

默认情况下，feign 不会收集任何指标。 但是，可以向任何 feign 客户端添加指标收集功能。 Metric Capabilities 提供了一流的 Metrics API，用户可以利用它来深入了解请求/响应生命周期。

 Dropwizard Metrics 4

```java
public class MyApp {
  public static void main(String[] args) {
    GitHub github = Feign.builder()
                         .addCapability(new Metrics4Capability())
                         .target(GitHub.class, "https://api.github.com");

    github.contributors("OpenFeign", "feign");
    // metrics will be available from this point onwards
  }
}
```

Dropwizard Metrics 5

```java
public class MyApp {
  public static void main(String[] args) {
    GitHub github = Feign.builder()
                         .addCapability(new Metrics5Capability())
                         .target(GitHub.class, "https://api.github.com");

    github.contributors("OpenFeign", "feign");
    // metrics will be available from this point onwards
  }
}
```

Micrometer

```java
public class MyApp {
  public static void main(String[] args) {
    GitHub github = Feign.builder()
                         .addCapability(new MicrometerCapability())
                         .target(GitHub.class, "https://api.github.com");

    github.contributors("OpenFeign", "feign");
    // metrics will be available from this point onwards
  }
}
```

### 4.9、静态和默认方法

Feign 所针对的接口可能具有静态或默认方法（如果使用 Java 8+）。这些允许 Feign 客户端包含未由底层 API 明确定义的逻辑。例如，静态方法可以轻松指定常见的客户端构建配置；默认方法可用于组合查询或定义默认参数。

```java
interface GitHub {
  @RequestLine("GET /repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@Param("owner") String owner, @Param("repo") String repo);

  @RequestLine("GET /users/{username}/repos?sort={sort}")
  List<Repo> repos(@Param("username") String owner, @Param("sort") String sort);

  default List<Repo> repos(String owner) {
    return repos(owner, "full_name");
  }

  /**
   * Lists all contributors for all repos owned by a user.
   */
  default List<Contributor> contributors(String user) {
    MergingContributorList contributors = new MergingContributorList();
    for(Repo repo : this.repos(owner)) {
      contributors.addAll(this.contributors(user, repo.getName()));
    }
    return contributors.mergeResult();
  }

  static GitHub connect() {
    return Feign.builder()
                .decoder(new GsonDecoder())
                .target(GitHub.class, "https://api.github.com");
  }
}
```

### 4.10、通过 CompletableFuture 异步执行

Feign 10.8 引入了一个新的构建器 AsyncFeign，它允许方法返回 CompletableFuture 实例。

```java
interface GitHub {
  @RequestLine("GET /repos/{owner}/{repo}/contributors")
  CompletableFuture<List<Contributor>> contributors(@Param("owner") String owner, @Param("repo") String repo);
}

public class MyApp {
  public static void main(String... args) {
    GitHub github = AsyncFeign.asyncBuilder()
                         .decoder(new GsonDecoder())
                         .target(GitHub.class, "https://api.github.com");

    // Fetch and print a list of the contributors to this library.
    CompletableFuture<List<Contributor>> contributors = github.contributors("OpenFeign", "feign");
    for (Contributor contributor : contributors.get(1, TimeUnit.SECONDS)) {
      System.out.println(contributor.login + " (" + contributor.contributions + ")");
    }
  }
}
```

初始实现包括 2 个异步客户端：

- `AsyncClient.Default`
- `AsyncApacheHttp5Client`