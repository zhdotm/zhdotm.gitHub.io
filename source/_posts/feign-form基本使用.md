---
title: feign-form基本使用
date: 2021-11-09 08:12:43
tags: Feign
categories: Feign
---

# feign-form基本使用

此模块添加了对编码 application/x-www-form-urlencoded 和 multipart/form-data 表单的支持。

## 1、添加依赖

包含对您的应用程序的依赖项：

### 1.1、**Maven**:

```xml
<dependencies>
  ...
  <dependency>
    <groupId>io.github.openfeign.form</groupId>
    <artifactId>feign-form</artifactId>
    <version>3.8.0</version>
  </dependency>
  ...
</dependencies>
```

### 1.2、**Gradle**:

```xml
compile 'io.github.openfeign.form:feign-form:3.8.0'
```

## 2、要求

feign-form 扩展依赖于 OpenFeign 及其具体版本：

- 3.5.0 之前的所有 feign-form 版本都适用于 OpenFeign 9.* 版本；
- 从 feign-form 的 3.5.0 版开始，该模块适用于 OpenFeign 10.1.0 及更高版本。

重要提示：没有向后兼容性，也没有任何保证 3.5.0 之后的 feign-form 版本与 10.* 之前的 OpenFeign 一起使用。 OpenFeign 在第 10 个版本中被重构，所以最好的方法 - 使用最新的 OpenFeign 和 feign-form 版本。

注意：

- spring-cloud-openfeign 在 v2.0.3.RELEASE 之前使用 OpenFeign 9.*，之后使用 10.*。反正这个依赖已经有合适的feign-form版本了，看依赖pom，不需要单独指定；
- spring-cloud-starter-feign 是一个已弃用的依赖项，它始终使用 OpenFeign 的 9.* 版本。

## 3、用法

像这样将 FormEncoder 添加到 Feign.Builder 中：

```java
SomeApi github = Feign.builder()
                      .encoder(new FormEncoder())
                      .target(SomeApi.class, "http://api.some.org");
```

此外，您可以像这样装饰现有的编码器，例如 JsonEncoder：

```java
SomeApi github = Feign.builder()
                      .encoder(new FormEncoder(new JacksonEncoder()))
                      .target(SomeApi.class, "http://api.some.org");
```

并一起使用它们：

```java
interface SomeApi {

  @RequestLine("POST /json")
  @Headers("Content-Type: application/json")
  void json (Dto dto);

  @RequestLine("POST /form")
  @Headers("Content-Type: application/x-www-form-urlencoded")
  void from (@Param("field1") String field1, @Param("field2") String[] values);
}
```

您可以通过 Content-Type 标头指定两种类型的编码形式。

application/x-www-form-urlencoded

```java
interface SomeApi {

  @RequestLine("POST /authorization")
  @Headers("Content-Type: application/x-www-form-urlencoded")
  void authorization (@Param("email") String email, @Param("password") String password);

  // Group all parameters within a POJO
  @RequestLine("POST /user")
  @Headers("Content-Type: application/x-www-form-urlencoded")
  void addUser (User user);

  class User {

    Integer id;

    String name;
  }
}
```

multipart/form-data

```java
interface SomeApi {

  // File parameter
  @RequestLine("POST /send_photo")
  @Headers("Content-Type: multipart/form-data")
  void sendPhoto (@Param("is_public") Boolean isPublic, @Param("photo") File photo);

  // byte[] parameter
  @RequestLine("POST /send_photo")
  @Headers("Content-Type: multipart/form-data")
  void sendPhoto (@Param("is_public") Boolean isPublic, @Param("photo") byte[] photo);

  // FormData parameter
  @RequestLine("POST /send_photo")
  @Headers("Content-Type: multipart/form-data")
  void sendPhoto (@Param("is_public") Boolean isPublic, @Param("photo") FormData photo);

  // Group all parameters within a POJO
  @RequestLine("POST /send_photo")
  @Headers("Content-Type: multipart/form-data")
  void sendPhoto (MyPojo pojo);

  class MyPojo {

    @FormProperty("is_public")
    Boolean isPublic;

    File photo;
  }
}
```

在上面的示例中，sendPhoto 方法使用 photo 参数使用三种不同的受支持类型。

- File 将使用 File 的扩展名来检测 Content-Type；
- byte[] 将使用 application/octet-stream 作为 Content-Type；
- FormData 将使用 FormData 的 Content-Type 和 fileName；
- 用于分组参数（包括上述类型）的客户端自定义 POJO。

FormData 是一个自定义对象，它包装了一个 byte[] 并定义了一个 Content-Type 和 fileName，如下所示：

```java
  FormData formData = new FormData("image/png", "filename.png", myDataAsByteArray);
  someApi.sendPhoto(true, formData);
```

## 4、Spring MultipartFile 和 Spring Cloud Netflix @FeignClient 支持

您还可以将表单编码器与 Spring MultipartFile 和 @FeignClient 一起使用。 将依赖项包含到项目的 pom.xml 文件中：

```xml
<dependencies>
  <dependency>
    <groupId>io.github.openfeign.form</groupId>
    <artifactId>feign-form</artifactId>
    <version>3.8.0</version>
  </dependency>
  <dependency>
    <groupId>io.github.openfeign.form</groupId>
    <artifactId>feign-form-spring</artifactId>
    <version>3.8.0</version>
  </dependency>
</dependencies>
```

```java
@FeignClient(
    name = "file-upload-service",
    configuration = FileUploadServiceClient.MultipartSupportConfig.class
)
public interface FileUploadServiceClient extends IFileUploadServiceClient {

  public class MultipartSupportConfig {

    @Autowired
    private ObjectFactory<HttpMessageConverters> messageConverters;

    @Bean
    public Encoder feignFormEncoder () {
      return new SpringFormEncoder(new SpringEncoder(messageConverters));
    }
  }
}
```

或者，如果您不需要 Spring 的标准编码器：

```java
@FeignClient(
    name = "file-upload-service",
    configuration = FileUploadServiceClient.MultipartSupportConfig.class
)
public interface FileUploadServiceClient extends IFileUploadServiceClient {

  public class MultipartSupportConfig {

    @Bean
    public Encoder feignFormEncoder () {
      return new SpringFormEncoder();
    }
  }
}
```

感谢 tf-haotri-pham 的特性，它利用了 Apache commons-fileupload 库，处理多部分响应的解析。正文数据部分作为字节数组保存在内存中。 要使用此功能，请在解码器的消息转换器列表中包含 SpringManyMultipartFilesReader，并让 Feign 客户端返回一个 MultipartFile 数组：

```java
@FeignClient(
    name = "${feign.name}",
    url = "${feign.url}"
    configuration = DownloadClient.ClientConfiguration.class
)
public interface DownloadClient {

  @RequestMapping("/multipart/download/{fileId}")
  MultipartFile[] download(@PathVariable("fileId") String fileId);

  class ClientConfiguration {

    @Autowired
    private ObjectFactory<HttpMessageConverters> messageConverters;

    @Bean
    public Decoder feignDecoder () {
      List<HttpMessageConverter<?>> springConverters =
            messageConverters.getObject().getConverters();

      List<HttpMessageConverter<?>> decoderConverters =
            new ArrayList<HttpMessageConverter<?>>(springConverters.size() + 1);

      decoderConverters.addAll(springConverters);
      decoderConverters.add(new SpringManyMultipartFilesReader(4096));

      HttpMessageConverters httpMessageConverters = new HttpMessageConverters(decoderConverters);

      return new SpringDecoder(new ObjectFactory<HttpMessageConverters>() {

        @Override
        public HttpMessageConverters getObject() {
          return httpMessageConverters;
        }
      });
    }
  }
}
```