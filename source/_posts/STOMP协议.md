---
title: STOMP协议
date: 2021-11-09 08:12:43
tags: [网络协议, STOMP]
categories: [网络协议, STOMP]
---

# STOMP协议

# STOMP 协议规范，版本 1.2

## 概述

### 背景

STOMP 源于需要从 Ruby、Python 和 Perl 等脚本语言连接到企业消息代理。在这样的环境中，通常执行逻辑上简单的操作，例如“可靠地发送单个消息并断开连接”或“使用给定目的地上的所有消息”。

它是其他开放消息传递协议（如 AMQP）和 JMS 代理（如 OpenWire）中使用的特定于实现的线路协议的替代方案。它通过覆盖一小部分常用消息传递操作而不是提供全面的消息传递 API 来区分自己。

最近，STOMP 已经成熟为一种协议，该协议可以在其现在提供的线级功能方面超越这些简单的用例，但仍保持其简单性和互操作性的核心设计原则。

### 协议概述

STOMP 是一种基于帧的协议，其帧以 HTTP 为模型。一个帧由一个命令、一组可选标题和一个可选正文组成。 STOMP 是基于文本的，但也允许传输二进制消息。 STOMP 的默认编码是 UTF-8，但它支持消息正文的替代编码规范。

STOMP 服务器被建模为一组可以发送消息的目的地。 STOMP 协议将目的地视为不透明字符串，它们的语法是特定于服务器实现的。此外，STOMP 没有定义目的地的交付语义应该是什么。目的地的传递或“消息交换”语义可能因服务器而异，甚至因目的地而异。这允许服务器在他们可以用 STOMP 支持的语义上进行创造性的工作。

STOMP 客户端是一个用户代理，可以在两种（可能同时）模式下运行：

- 作为生产者，通过 SEND 帧将消息发送到服务器上的目的地
- 作为消费者，为给定的目的地发送一个订阅帧，并从服务器接收消息作为 MESSAGE 帧。

### 协议的变化

STOMP 1.2 主要向后兼容 STOMP 1.1。只有两个不兼容的更改：

- 现在可以使用回车加换行而不是仅换行来结束帧行
- 消息确认已被简化，现在使用专用标头

除此之外，STOMP 1.2 没有引入任何新特性，而是着重于阐明规范的一些领域，例如：

- 重复的帧头条目
- 使用 content-length 和 content-type 标头
- 需要服务器支持 STOMP 框架
- 持续连接
- 订阅和transaction标识符的范围和唯一性
- RECEIPT 帧相对于先前帧的意义

### 设计理念

推动 STOMP 设计的主要理念是简单性和互操作性。

STOMP 被设计为一种轻量级协议，易于在客户端和服务器端以多种语言实现。这尤其意味着，对服务器架构的约束并不多，而且许多特性（例如目标命名和可靠性语义）是特定于实现的。

在本规范中，我们将注意 STOMP 1.2 未明确定义的服务器功能。您应该查阅 STOMP 服务器的文档以了解这些功能的实现特定细节。

### 一致性

本文档中的关键词“MUST”、“MUST NOT”、“REQUIRED”、“SHALL”、“SHALL NOT”、“SHOULD”、“SHOULD NOT”、“RECOMMENDED”、“MAY”和“OPTIONAL”是按照 RFC 2119 中的描述进行解释。

实现可能会对不受约束的输入施加特定于实现的限制，例如以防止拒绝服务攻击、防止内存不足或解决特定于平台的限制。

本规范定义的一致性类是 STOMP 客户端和 STOMP 服务器。

## STOMP帧

STOMP 是一种基于帧的协议，它在下面假定了一个可靠的 2-way 流网络协议（例如 TCP）。客户端和服务器将使用通过流发送的 STOMP 帧进行通信。框架的结构如下所示：

```java
COMMAND
header1:value1
header2:value2

Body^
```

该帧以一个以行尾 (EOL) 结尾的命令字符串开始，它由一个可选的回车（octet 13）和一个必需的换行符（octet 10）组成。命令后面是 <key>:<value> 格式的零个或多个标头条目。每个报头条目都由一个 EOL 终止。空行（即额外的 EOL）表示标题的结尾和正文的开头。正文之后是 NULL 八位字节。本文档中的示例将使用 ASCII 中的 ^@、control-@ 来表示 NULL 八位字节。 NULL 八位字节可以有选择地跟随多个 EOL。有关如何解析 STOMP 帧的更多详细信息，请参阅本文档的Augmented BNF 部分。

本文档中引用的所有命令和标题名称均区分大小写。

### 值编码

命令和标头以 UTF-8 编码。除 CONNECT 和 CONNECTED 帧之外的所有帧也将转义在生成的 UTF-8 编码标头中找到的任何回车、换行或冒号。

需要转义以允许标题键和值包含那些作为值分隔八位字节的帧标题。 CONNECT 和 CONNECTED 帧不会转义回车、换行或冒号八位字节，以保持与 STOMP 1.0 的向后兼容。

C 样式字符串文字转义用于对在 UTF-8 编码标头中找到的任何回车、换行或冒号进行编码。解码帧头时，必须应用以下转换：

- `\r` (octet 92 and 114) 翻译成回车 (octet 13)
- `\n` (octet 92 and 110) 转换为换行 (octet 10)
- `\c` (octet 92 and 99) 翻译成 `:` (octet 58)
- `\\` (octet 92 and 92) 翻译成 `\` (octet 92)

未定义的转义序列如 \t（octet 92 and 116）必须被视为致命的协议错误。相反，在编码帧头时，必须应用逆向转换。

STOMP 1.0 规范包括许多在标头中填充的示例帧，并且实现了许多服务器和客户端来修剪或填充标头值。如果应用程序想要发送不应被修剪的标头，这会导致问题。在 STOMP 1.2 中，客户端和服务器绝不能用空格修剪或填充标头。

### Body

只有 SEND、MESSAGE 和 ERROR 帧可能有正文。所有其他帧不得有主体。

### 标准头

对于大多数帧，可以使用某些标题并且具有特殊含义。

#### Header content-length

所有的帧都可以包含一个内容长度的头部。此标头是消息正文长度的八位字节计数。如果包含内容长度标头，则必须读取此八位字节数，无论正文中是否存在 NULL 八位字节。该帧仍需要以 NULL 八位字节结束。

如果存在帧体，则 SEND、MESSAGE 和 ERROR 帧应该包含内容长度标头以简化帧解析。如果帧体包含 NULL 八位字节，则帧必须包含内容长度头。

#### Header content-type

如果存在帧体，则 SEND、MESSAGE 和 ERROR 帧应该包含一个内容类型的头，以帮助帧的接收者解释其体。如果设置了 content-type 标头，它的值必须是描述正文格式的 MIME 类型。否则，接收者应该将主体视为二进制 blob。

以 text/ 开头的 MIME 类型的隐含文本编码是 UTF-8。如果您使用具有不同编码的基于文本的 MIME 类型，那么您应该将 ;charset=<encoding> 附加到 MIME 类型。例如，如果您以 UTF-16 编码发送 HTML 正文，则应使用 text/html;charset=utf-16 。 ;charset=<encoding> 也应该附加到任何可以解释为文本的非文本/ MIME 类型。一个很好的例子是 UTF-8 编码的 XML。它的内容类型应该设置为 application/xml;charset=utf-8

所有 STOMP 客户端和服务器必须支持 UTF-8 编码和解码。因此，为了在异构计算环境中实现最大的互操作性，建议使用 UTF-8 对基于文本的内容进行编码。

#### Header receipt

除了 CONNECT 之外的任何客户端框架都可以指定具有任意值的接收标头。这将导致服务器使用 RECEIPT 帧确认客户端帧的处理（有关更多详细信息，请参阅 RECEIPT 帧）。

```java
SEND
destination:/queue/a
receipt:message-12345

hello queue a^@
```

### 重复的头条目

由于消息系统可以按照存储和转发拓扑进行组织，类似于 SMTP，因此消息在到达消费者之前可能会遍历多个消息服务器。 STOMP 服务器可以通过在消息中添加标头或在消息中就地修改标头来“更新”标头值。

如果客户端或服务器收到重复的帧头条目，则只有第一个头条目应该用作头条目的值。后续值仅用于维护标头状态更改的历史记录，可以忽略。

例如，如果客户端收到：

```java
MESSAGE
foo:World
foo:Hello

^@
```

foo 头的值就是 World。

### 大小限制

为了防止恶意客户端利用服务器中的内存分配，服务器可以设置最大限制：

- 单个帧中允许的帧头数
- Header行的最大长度
- 帧体的最大尺寸

如果超过这些限制，服务器应该向客户端发送一个 ERROR 帧，然后关闭连接。

### 连接延迟

STOMP 服务器必须能够支持快速连接和断开连接的客户端。

这意味着服务器可能只允许关闭的连接在连接重置之前短暂停留。

因此，在套接字重置之前，客户端可能不会收到服务器发送的最后一帧（例如，错误帧或响应断开帧的接收帧）。

### 连接

STOMP 客户端通过发送 CONNECT 帧来启动与服务器的流或 TCP 连接：

```java
CONNECT
accept-version:1.2
host:stomp.github.org

^@
```

如果服务器接受连接尝试，它将以 CONNECTED 帧响应：

```java
CONNECTED
version:1.2

^@
```

服务器可以拒绝任何连接尝试。服务器应该用一个 ERROR 帧来响应，解释连接被拒绝的原因，然后关闭连接。

### CONNECT或STOMP帧

STOMP 服务器必须以与 CONNECT 帧相同的方式处理 STOMP 帧。 STOMP 1.2 客户端应该继续使用 CONNECT 命令来保持与 STOMP 1.0 服务器的向后兼容。

使用 STOMP 帧而不是 CONNECT 帧的客户端将只能连接到 STOMP 1.2 服务器（以及一些 STOMP 1.1 服务器），但优点是协议嗅探器/鉴别器将能够将 STOMP 连接与HTTP 连接。

STOMP 1.2 客户端必须设置以下标头：

- `accept-version` : 客户端支持的 STOMP 协议版本。有关更多详细信息，请参阅协议协商
- `host` : 客户端希望连接的虚拟主机的名称。建议客户端将其设置为建立套接字所针对的主机名，或他们选择的任何名称。如果此标头与已知虚拟主机不匹配，则支持虚拟主机的服务器可以选择默认虚拟主机或拒绝连接。

STOMP 1.2 客户端可以设置以下标头：

- `login` : 用于针对安全 STOMP 服务器进行身份验证的用户标识符。
- `passcode` : 用于对安全 STOMP 服务器进行身份验证的密码。
- `heart-beat` : 心跳设置。

### 连接帧

STOMP 1.2 服务器必须设置以下标头：

- `version` : 会话将使用的 STOMP 协议的版本。有关更多详细信息，请参阅协议协商。

STOMP 1.2 服务器可以设置以下标头：

- `heart-beat` : 心跳设置。

- `session` : 唯一标识会话的会话标识符。

- `server` : 包含有关 STOMP 服务器的信息的字段。该字段必须包含一个服务器名称字段，并且可以跟随着由空格八位字节分隔的可选注释字段。

  server-name 字段由一个名称标记和一个可选的版本号标记组成。

  `server = name ["/" version] *(comment)`

  例子:

  `server:Apache/1.3.9`

### 协议交互

从 STOMP 1.1 开始，CONNECT 帧必须包含 accept-version 标头。它应该设置为客户端支持的递增 STOMP 协议版本的逗号分隔列表。如果缺少accept-version 头，则表示客户端仅支持1.0 版协议。

将用于会话其余部分的协议将是客户端和服务器共有的最高协议版本。

例如，如果客户端发送：

```java
CONNECT
accept-version:1.0,1.1,2.0
host:stomp.github.org

^@
```

服务器将使用与客户端相同的协议的最高版本进行响应：

```java
CONNECTED
version:1.1

^@
```

如果客户端和服务器不共享任何公共协议版本，则服务器必须以类似于以下的 ERROR 帧响应，然后关闭连接：

```java
ERROR
version:1.2,2.1
content-type:text/plain

Supported protocol versions are 1.2 2.1^@
```

### 心跳

可以选择使用心跳来测试底层 TCP 连接的健康状况，并确保远程端处于活动状态并正常运行。

为了启用心跳，每一方都必须声明它可以做什么以及希望另一方做什么。这发生在 STOMP 会话的最开始时，通过向 CONNECT 和 CONNECTED 帧添加心跳标头。

使用时，心跳头必须包含两个用逗号分隔的正整数。

第一个数字代表帧的发送者可以做什么（传出心跳）：

- 0 表示不能发送心跳

- 否则它是它可以保证的心跳之间的最小毫秒数 第二个数字代表帧的发送者想要得到什么（传入的心跳）：

  - 0 表示它不想接收心跳

  - 否则它是心跳之间所需的毫秒数

心跳报头是可选的。丢失的心跳报头必须以与“heart-beat:0,0”报头相同的方式处理，即：一方不能发送也不想接收心跳。

心跳报头提供了足够的信息，以便每一方都可以查明是否可以使用心跳、在哪个方向以及以哪个频率使用。

更正式地说，初始帧如下所示：

```java
CONNECT
heart-beat:<cx>,<cy>

CONNECTED
heart-beat:<sx>,<sy>
```

对于从客户端到服务器的心跳：

- 如果 <cx> 为 0（客户端无法发送心跳）或 <sy> 为 0（服务器不想接收心跳），则不会有
- 否则，每 MAX(<cx>,<sy>) 毫秒就会有一次心跳

在另一个方向上，<sx> 和 <cy> 的用法相同。

关于心跳本身，通过网络连接接收到的任何新数据都表明远程端处于活动状态。在给定的方向上，如果每 <n> 毫秒需要一次心跳：

- 发送方必须至少每 <n> 毫秒通过网络连接发送新数据
- 如果发送方没有真正的 STOMP 帧要发送，它必须发送一个行尾（EOL）
- 如果在至少 <n> 毫秒的时间窗口内，接收器没有收到任何新数据，它可以认为连接已死
- 由于时序不准确，接收器应该容忍并考虑误差容限

## 客户端帧

客户端可以发送不在这个列表中的帧，但是对于这样的帧，STOMP 1.2 服务器可以响应一个 ERROR 帧，然后关闭连接。

- [`SEND`](https://stomp.github.io/stomp-specification-1.2.html#SEND)
- [`SUBSCRIBE`](https://stomp.github.io/stomp-specification-1.2.html#SUBSCRIBE)
- [`UNSUBSCRIBE`](https://stomp.github.io/stomp-specification-1.2.html#UNSUBSCRIBE)
- [`BEGIN`](https://stomp.github.io/stomp-specification-1.2.html#BEGIN)
- [`COMMIT`](https://stomp.github.io/stomp-specification-1.2.html#COMMIT)
- [`ABORT`](https://stomp.github.io/stomp-specification-1.2.html#ABORT)
- [`ACK`](https://stomp.github.io/stomp-specification-1.2.html#ACK)
- [`NACK`](https://stomp.github.io/stomp-specification-1.2.html#NACK)
- [`DISCONNECT`](https://stomp.github.io/stomp-specification-1.2.html#DISCONNECT)

### SEND

SEND 帧将消息发送到消息系统中的目的地。它有一个 REQUIRED 标头，即目的地，指示将消息发送到何处。 SEND 帧的主体是要发送的消息。例如：

```java
SEND
destination:/queue/a
content-type:text/plain

hello queue a
^@
```

这会向名为 /queue/a 的目的地发送一条消息。请注意，STOMP 将此目的地视为不透明字符串，并且目的地名称不假定传递语义。您应该查阅 STOMP 服务器的文档以了解如何构造一个目标名称，该名称为您提供应用程序所需的传递语义。

消息的可靠性语义也是特定于服务器的，将取决于正在使用的目标值和其他消息头，例如事务头或其他服务器特定的消息头。

SEND 支持允许事务发送的事务标头。

如果正文存在，SEND 帧应该包括一个[`content-length`](https://stomp.github.io/stomp-specification-1.2.html#Header_content-length) 标头和一个[`content-type`](https://stomp.github.io/stomp-specification-1.2.html#Header_content-type) 标头。

应用程序可以向 SEND 帧添加任意用户定义的标头。用户定义的标头通常用于允许消费者使用订阅帧上的选择器根据应用程序定义的标头过滤消息。用户定义的头必须在 MESSAGE 帧中传递。

如果服务器由于任何原因无法成功处理 SEND 帧，则服务器必须向客户端发送一个 ERROR 帧，然后关闭连接。

### SUBSCRIBE

SUBSCRIBE 帧用于注册以侦听给定的目的地。与 SEND 帧一样，SUBSCRIBE 帧需要一个目的地标头，指示客户端想要订阅的目的地。在订阅目标上收到的任何消息将作为 MESSAGE 帧从服务器传递到客户端。 ack 头控制消息确认模式。

例子：

```java
SUBSCRIBE
id:0
destination:/queue/foo
ack:client

^@
```

如果服务器无法成功创建订阅，则服务器必须向客户端发送一个 ERROR 帧，然后关闭连接。

STOMP 服务器可以支持额外的特定于服务器的标头来定制订阅的交付语义。有关详细信息，请参阅服务器的文档。

#### SUBSCRIBE id Header

由于单个连接可以与服务器有多个开放订阅，因此帧中必须包含一个 id 标头以唯一标识订阅。 id 头允许客户端和服务器将后续的 MESSAGE 或 UNSUBSCRIBE 帧与原始订阅相关联。

在同一个连接中，不同的订阅必须使用不同的订阅标识符。

#### SUBSCRIBE ack Header

ack 标头的有效值为 auto、client 或 client-individual。如果未设置标头，则默认为自动。

当 ack 模式为 auto 时，客户端不需要为它收到的消息发送服务器 ACK 帧。服务器将在将消息发送给客户端后立即假定客户端已收到该消息。这种确认模式可能会导致传输到客户端的消息被丢弃。

当 ack 模式为客户端时，客户端必须为它处理的消息发送服务器 ACK 帧。如果在客户端发送消息的 ACK 帧之前连接失败，服务器将假定消息尚未处理，并可以将消息重新传递给另一个客户端。客户端发送的 ACK 帧将被视为累积确认。这意味着确认对 ACK 帧中指定的消息以及在 ACK 消息之前发送到订阅的所有消息进行操作。

如果客户端没有处理一些消息，它应该发送 NACK 帧来告诉服务器它没有使用这些消息。

当 ack 模式为 client-individual 时，除了客户端发送的 ACK 或 NACK 帧不累积之外，确认的操作与客户端确认模式相同。这意味着后续消息的 ACK 或 NACK 帧不得导致先前消息得到确认。

### UNSUBSCRIBE

UNSUBSCRIBE 帧用于删除现有订阅。一旦订阅被删除，STOMP 连接将不再接收来自该订阅的消息。

由于单个连接可以与服务器有多个开放订阅，因此帧中必须包含一个 id 标头以唯一标识要删除的订阅。此标头必须与现有订阅的订阅标识符匹配。

例子：

```java
UNSUBSCRIBE
id:0

^@
```

### ACK

ACK 用于使用客户端或客户端个人确认来确认来自订阅的消息的消耗。在通过 ACK 确认消息之前，不会认为从此类订阅接收到的任何消息已被消费。

ACK 帧必须包含一个与被确认的 MESSAGE 的 ack 头匹配的 id 头。可选地，可以指定一个事务头，表明消息确认应该是命名事务的一部分。

```java
ACK
id:12345
transaction:tx1

^@
```

### NACK

NACK 是 ACK 的反义词。它用于告诉服务器客户端没有消费该消息。然后，服务器可以将消息发送到不同的客户端、丢弃它或将其放入死信队列。确切的行为是特定于服务器的。

NACK 采用与 ACK 相同的标头：id（必需）和交易（可选）。

NACK 应用于单个消息（如果订阅的 ack 模式是客户端个人）或应用于之前发送但尚未确认或 NACK 的所有消息（如果订阅的确认模式是客户端）。

### BEGIN

BEGIN 用于启动事务。这种情况下的交易适用于发送和确认 - 在交易期间发送或确认的任何消息都将根据交易进行原子处理。

```java
BEGIN
transaction:tx1

^@
```

事务标头是必需的，事务标识符将用于 SEND、COMMIT、ABORT、ACK 和 NACK 帧以将它们绑定到命名事务。在同一个连接中，不同的事务必须使用不同的事务标识符。

如果客户端发送 DISCONNECT 帧或 TCP 连接因任何原因失败，则任何尚未提交的已启动事务都将隐式中止。

### COMMIT

COMMIT 用于提交正在进行的事务。

```java
COMMIT
transaction:tx1

^@
```

事务头是必需的，并且必须指定要提交的事务的标识符。

### ABORT

ABORT 用于回滚正在进行的事务。

```java
ABORT
transaction:tx1

^@
```

事务头是必需的，并且必须指定要中止的事务的标识符。

### DISCONNECT

客户端可以随时通过关闭套接字与服务器断开连接，但不能保证先前发送的帧已被服务器接收。要进行正常关闭，客户端确保服务器已收到所有先前的帧，客户端应该：

1. 发送带有接收标头集的 DISCONNECT 帧。例子：

   ```
   DISCONNECT
   receipt:77
   ^@
   ```

2. 等待 RECEIPT 帧对 DISCONNECT 的响应。例子：

   ```
   RECEIPT
   receipt-id:77
   ^@
   ```

3. 关闭套接字。

请注意，如果服务器过快地关闭套接字的末尾，客户端可能永远不会收到预期的 RECEIPT 帧。有关更多信息，请参阅连接延迟部分。

发送 DISCONNECT 帧后，客户端不得再发送任何帧。

## 服务端帧

服务器有时会向客户端发送帧（除了初始的 CONNECTED 帧）。这些帧可能是以下之一：

- [`MESSAGE`](https://stomp.github.io/stomp-specification-1.2.html#MESSAGE)
- [`RECEIPT`](https://stomp.github.io/stomp-specification-1.2.html#RECEIPT)
- [`ERROR`](https://stomp.github.io/stomp-specification-1.2.html#ERROR)

### MESSAGE

MESSAGE 帧用于将消息从订阅传送到客户端。

MESSAGE 帧必须包含一个目的地标头，指示消息被发送到的目的地。如果消息是使用 STOMP 发送的，则此目标标头应该与相应 SEND 帧中使用的标头相同。

MESSAGE 帧还必须包含一个 message-id 标头，该标头具有该消息的唯一标识符和一个与接收消息的订阅标识符匹配的订阅标头。

如果从需要显式确认的订阅接收消息（客户端或客户端个人模式），则 MESSAGE 帧还必须包含具有任意值的 ack 标头。该报头将用于将消息与后续 ACK 或 NACK 帧相关联。

帧体包含消息的内容：

```java
MESSAGE
subscription:0
message-id:007
destination:/queue/a
content-type:text/plain

hello queue a^@
```

如果正文存在，MESSAGE 帧应该包括一个[`content-length`](https://stomp.github.io/stomp-specification-1.2.html#Header_content-length)标头和一个[`content-type`](https://stomp.github.io/stomp-specification-1.2.html#Header_content-type)标头。

MESSAGE 帧还将包括所有用户定义的头，这些头在消息发送到目的地时出现，除了可能被添加到帧中的服务器特定头。查阅您的服务器的文档以找出它添加到消息中的特定于服务器的标头。

### RECEIPT

一旦服务器成功处理了请求接收的客户端帧，就会从服务器向客户端发送一个 RECEIPT 帧。一个 RECEIPT 帧必须包含首部receipt-id，其中的值是作为接收的帧中的接收首部的值。

```java
RECEIPT
receipt-id:message-12345

^@
```

RECEIPT 帧是对相应客户端帧已被服务器处理的确认。由于 STOMP 是基于流的，因此接收也是服务器已接收到所有先前帧的累积确认。然而，这些先前的帧可能还没有被完全处理。如果客户端断开连接，先前收到的帧应该继续由服务器处理。

### ERROR

如果出现问题，服务器可能会发送 ERROR 帧。在这种情况下，它必须在发送 ERROR 帧后立即关闭连接。请参阅有关连接延迟的下一节。

ERROR 帧应该包含一个带有错误简短描述的消息头，并且主体可以包含更详细的信息（或者可以为空）。

```java
ERROR
receipt-id:message-12345
content-type:text/plain
content-length:170
message:malformed frame received

The message:
-----
MESSAGE
destined:/queue/a
receipt:message-12345

Hello queue a!
-----
Did not contain a destination header, which is REQUIRED
for message propagation.
^@
```

如果错误与客户端发送的特定帧有关，则服务器应该添加额外的标头以帮助识别导致错误的原始帧。例如，如果帧包含接收头，则错误帧应该设置接收标识头以匹配与错误相关的帧的接收头的值。

如果正文存在，则错误帧应该包括内容长度标头和内容类型标头。

## Frames and Headers

除了上面描述的标准头（内容长度、内容类型和接收）之外，这里是本规范中定义的每个帧必须或可以使用的所有头：

```java
CONNECT` or `STOMP
```

- REQUIRED: `accept-version`, `host`
- OPTIONAL: `login`, `passcode`, `heart-beat`

```java
CONNECTED
```

- REQUIRED: `version`
- OPTIONAL: `session`, `server`, `heart-beat`

```java
SEND
```

- REQUIRED: `destination`
- OPTIONAL: `transaction`

```java
SUBSCRIBE
```

- REQUIRED: `destination`, `id`
- OPTIONAL: `ack`

```java
UNSUBSCRIBE
```

- REQUIRED: `id`
- OPTIONAL: none

```java
ACK` or `NACK
```

- REQUIRED: `id`
- OPTIONAL: `transaction`

```java
BEGIN` or `COMMIT` or `ABORT
```

- REQUIRED: `transaction`
- OPTIONAL: none

```java
DISCONNECT
```

- REQUIRED: none
- OPTIONAL: `receipt`

```java
MESSAGE
```

- REQUIRED: `destination`, `message-id`, `subscription`
- OPTIONAL: `ack`

```java
RECEIPT
```

- REQUIRED: `receipt-id`
- OPTIONAL: none

```java
ERROR
```

- REQUIRED: none
- OPTIONAL: `message`

此外，SEND 和 MESSAGE 帧可以包含任意用户定义的头部，这些头部应该被认为是承载消息的一部分。此外，ERROR 帧应该包括额外的头以帮助识别导致错误的原始帧。

最后，STOMP 服务器可以使用额外的标头来访问诸如持久性或过期等功能。有关详细信息，请参阅服务器的文档。

## Augmented BNF

可以使用 HTTP/1.1 RFC 2616 中使用的 Backus-Naur Form (BNF) 语法更正式地描述 STOMP 会话。

```java
NULL                = <US-ASCII null (octet 0)>
LF                  = <US-ASCII line feed (aka newline) (octet 10)>
CR                  = <US-ASCII carriage return (octet 13)>
EOL                 = [CR] LF 
OCTET               = <any 8-bit sequence of data>

frame-stream        = 1*frame

frame               = command EOL
                      *( header EOL )
                      EOL
                      *OCTET
                      NULL
                      *( EOL )

command             = client-command | server-command

client-command      = "SEND"
                      | "SUBSCRIBE"
                      | "UNSUBSCRIBE"
                      | "BEGIN"
                      | "COMMIT"
                      | "ABORT"
                      | "ACK"
                      | "NACK"
                      | "DISCONNECT"
                      | "CONNECT"
                      | "STOMP"

server-command      = "CONNECTED"
                      | "MESSAGE"
                      | "RECEIPT"
                      | "ERROR"

header              = header-name ":" header-value
header-name         = 1*<any OCTET except CR or LF or ":">
header-value        = *<any OCTET except CR or LF or ":">
```