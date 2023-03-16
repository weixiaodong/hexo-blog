---
title: http2协议详解
date: 2023-03-15 14:38:46
tags:
---

## 一、 http2简介

http2是一个应用层传输协议，它是http协议的第二版吧，http2主要是基于google的spdy协议，spdy的关键技术被ht采纳，因此spdy的成员全程参与了http2协议制定过程。

<!--more-->

## 二、 http1.1 遇到的问题

### 2.1 http 1.1 协议繁琐

### 2.2 未充分利用tcp性能

### 2.3 延迟问题

### 2.4 http头阻塞问题

## 三、 http2 怎么解决上面问题

1. 对http头字段进行数据压缩，使用HPAKC算法
	http头字段多，所以采用压缩算法来进行压缩大小传输。
2. 修复http头阻塞问题
	这就与http2协议设计有关，主要涉及3个概念，流、消息、帧。
	流: stream, 存在于TCP连接中的一个虚拟连接通道。每个流通道都有一个整数ID标识，流还可以承载双向消息。
	消息： message，它是由数据帧构成。
	帧： data frame, http2中构成消息的最小单位。消息有一个或多个帧构成。数据帧有一个关键数据，这个帧属于哪个资源。也就是解析后可以直接组成资源。
3. 延迟问题
	也是跟http2协议设计相关。

http2的一些其他特性：

- 多路复用
- 服务端推送
- 消息报文是二进制格式

## 四、 http2特性介绍

- 二进制分帧
- 头部字段压缩
- 多路复用
- 服务端推送
- 流量控制和资源优先级

## 五、 http2协议内容详解

### 5.1 http2协议概述

http2是基于http语义，它提供了一种优化传输机制。 HTTP2支持http1.1所有核心特性，http2从以下几个方面进行了改进：

- http2中最小的传输单元叫做帧。 http2定义了很多类型的帧，每个帧服务于不同的目的。 例如HEADERS和DATA帧就构成了HTTP请求和应答的主体。 还有其他的比如window_update, push_promise等帧类型用于支持http2的其他特性。
- 多路复用。 每个http请求/应答在各自的流（stream）中完成数据交换。每个流都是相互独立。因此如果一个请求/应答阻塞或者速度很慢，也不会影响其他流中的请求/应答处理。 在一个TCP连接中就可以传输多个流数据而无需建立多个连接。
- 流量控制和优先级机制。这个可以有效利用流的多路复用机制。 流量控制可以确保只有接收者使用的数据会被传输。优先级机制可以确保重要的资源优先传输。
- HTTP增加了一种新的交互模式。即服务端可以推送应答给客户端。
- HEAD头数据压缩。因为HTTP头包含了大量冗余数据，HTTP2对这些数据进行了压缩，压缩后对于请求大小的影响显著，可以将多个请求压缩到一个包中。
- HTTP2数据采用二进制编码，而不是原来的文本格式数据。

HTTP2协议有两个标识符：

- 字符串“h2”标识使用了TLS的http2协议，标示运行于tls之上http2的地方
- 字符串“h2c”标识构建在tcp之上的http2协议，它是明文传输，该标识符用在http1.1的upgrade首部字段，以及其他需要标示运行于tcp之上的http2的地方。

### 5.2 http1.1 协议怎么升级到http2协议

#### 5.2.1 http升级

因为http1.1协议会存在很长时间，所以怎么把http1.1协议升级到http2协议就很关键了。

客户端发起一个http uri请求时，如果事前不知道下一跳是否支持http2，需要使用http upgrade机制。 客户端发起一个http 1.1请求，其中包含“h2c”的upgrade首部字段，该请求还必须包含一个http2-settings首部字段。

例如
```
GET / http/1.1
HOST: server.example.com
Connection: Upgrade,http2-settings
upgrade:h2c
http2-settings: <base64url encoding of http/2 settings payload>
```

- connection 连接方式是升级协议
- upgrade 升级到什么协议，例子中是升级到h2c

如果服务器不同意升级或者不支持upgrade升级，可以直接忽略，当成是http1.1请求和响应就好了。
如果服务器同意升级，响应格式为:
```
http/1.1 101 switching protocols
connection: upgrade
upgrade: h2c


[http/2 connection ...]
```

http响应升级的状态码是101. 在结束101响应的空行后，服务端可以开始发送http2数据帧了。



#### 5.2.2 https升级

以上都是没有涉及到https，现在http2几乎都是用到了https，这个又怎么升级呢？

前面讲到，运行在tls之上的http2协议标识未“h2”，客户端不能发送“h2c”协议标识，服务端自然也不能选择“h2c”协议响应。

多了tls之后，客户端和服务端双方需等到成功建立tls连接之后才能发送应用数。 而建立tls连接，双方本来就要进行协商，引入http2之后，要做的就是在原来协商机制中把http2的协商机制加进去。

http2是基于google的spdy协议开发的，在spdy中开发了一个名为NpN（next protpcol negotiation, 下一代协议协商）的tls扩展协议。 而在http2中，NPN被http2修改为ALPN（Application Layer Protocol Negotiation, 应用层协议协商），它也是tls的扩展协议，也就是说构筑在tls上的http2，使用ALPN扩展协议进行协商。


tlsv1.2 握手协议简略交互图

```

client                                                    server
clientHello                     -------->                 
                                                          serverHello
                                                          certificate*
                                                          serverkeyExchange*
                                                          certificateRequest*
                                <--------                 serverHelloDone
certificate*
clientKeyExchange
CertificateVerify*
[changeCipherSpec]
finished                         -------->                 
                                                          [changeCipherSpec]
                                 <--------                finished
Application Data                 <-------->               Application Data
```


### 5.3 http2协议内容解析

http2协议是由http1.1升级而来，http的语义不变，提供的功能没有变化，http方法、状态码、uri和header字段这些都没有变化。在http2中，传输数据时编码是不同的，与换行符分割文本的http1.1协议不同，http2中数据交互都被拆分为更小的消息和帧，而每个消息和帧都是用二进制格式来编码。帧是http2中最小数据单元。

#### 5.3.1 http2 数据中几个重要的概念-帧、消息、流

- 帧 frame: http2中最小通信数据单元，每个帧至少包含了一个标识（stream identifier简称stream id）该帧所属的流。
- 消息message： 消息由一个或多个帧组成。例如请求的消息和响应的消息。
- 流stream： 存在于http2连接中的一个虚拟连接通道，它是一个逻辑概念。流可以承载双向字节流，即是客户端和服务端可以进行双向通信的字节序列。每个流都有一个唯一的整数ID（stream identifier）标识，由发起流的一端分配给流。

单个http2连接可以包含多个同时打开的流，任何一个端点（客户端和服务端）都可以将多个流的消息进行传输。这也是多路复用关键所在。 一个tcp连接（http2连接建立在tcp连接之上）里可以发送若干个流（stream），每个流可以传输若干条消息（message），每个message由若干个二进制帧组成。

任何一端都可以关闭流，在流上发送消息的顺序很重要，最后接收端会把stream identifier（同一个流）相同的帧重新组装成完整的消息报文。特别是headers帧和data帧的顺序在语义上非常重要。

http2中连接connection、 流stream、 消息message、 帧frame的关系示意图如下：

![](/images/http2组成剖析.png)



#### 5.3.2 二进制分帧层(HTTP2格式框架)

![](/images/HTTP2格式框架.png)

从上图可以看出，http1.1是明文文本，而http2.0首部（headers）和数据消息主体（data）都是帧（frame）。 frame是http2协议中最小数据传输单元。

#### 5.3.3 帧frame的格式

一旦建立了http2连接，端点（endpoints）间就可以开始交换帧数据。

所有的帧数据都是以一个固定的9字节开头（frame payload之前），后面跟一个可变长度的有效负载frame payload，这个可变长度的长度值由字段length来表示。

```
+-----------------------------------------------+
|                 Length (24)                   |
+---------------+---------------+---------------+
|   Type (8)    |   Flags (8)   |
+-+-------------+---------------+-------------------------------+
|R|                 Stream Identifier (31)                      |
+=+=============================================================+
|                   Frame Payload (0...)                      ...
+---------------------------------------------------------------+

HTTP Frame {
  Length (24),
  Type (8),

  Flags (8),

  Reserved (1),
  Stream Identifier (31),

  Frame Payload (..),
}
```

- length: 24个bit的无符号整数，用来表示frame payload的长度占用字节数。 这个length值的范围为 0-16384（2^14）。 触发接收方设置了一个更大的settings_max_frame_size.
- type: 定义frame的类型，8bite表示帧类型决定了帧的格式和语义。实现的话必须忽略或抛弃未知类型的帧
- flags： 为帧frame类型保留的8bit的布尔值，这个标志用于特定的帧frame类型语义。 如果这个字段没有被标识未特定帧类型语义，那么接收时必须被忽略，并且发送时不设置（0x0）
- R(Reserved): 一个保留的1bit字段，这个字段语义未定义。发送时必须保持未设置（0x0），接收时忽略。
- stream identifier: 流标识，31bit的无符号整数，值0x0保留给与整个连接相关联的帧，而不是单个流。
- frame payload： 内容主体，由帧的类型决定。

#### 5.3.4 有哪些帧类型type

上一小节的type字段，即帧类型，在http2中共分为10种类型：
每一种类型帧都由一个8位类型代码来识别。每种帧类型在建立和管理整个连接或者单个数据流时都有不同的作用


| type | code(type值) | 说明 |
| ----  |----  | ----  |
| Data | 0x00 | 数据帧（type = 0x00）, 内容主题信息frame payload。 比如一个或多个data帧可用于传输请求或响应的信息内容|
| headers | 0x01| 头帧 (type = 0x01), 用于打开一个流，另外还携带一个首部的块片段 |
| priority | 0x02 | 优先级帧（type = 0x02）, 在rfc9113中这个字段已弃用 |
| rst_stream | 0x03| 流终止帧 （type = 0x03）, 允许立即终止一个流。发送rst_stream表示请求取消流或表明发送错误的情况 |
| settings | 0x04 | 设置帧 (type = 0x04), 设置两端连接方式的配置参数 |
| push_promi | 0x05 | 推送帧 (type = 0x05), 服务端的推送，告诉对端打算推送数据给你了 |
| ping | 0x06 | ping帧（type = 0x06), 确定一个空闲连接是否仍然可用，也可以测量端点间往返时间（RTT）。 ping帧可以从任意端点发出 |
| goaway | 0x07 | goaway帧 (type = 0x07), 用于发起关闭连接的请求，或发出严重错误的信号。 goaway 允许端点优雅的停止接收新流，同时仍然完成对先前建立的流的处理。 |
| window_up | 0x08 | window_update帧(type = 0x08),用于实现流控，可以作用在单独的某个流上（指定具体的stream identifier）， 也可以作用在整个连接上 (stream identifier为0x0). 只有data帧会受到流控的影响 |
| continuation | 0x09 | 延续帧（type = 0x09）, 用于继续传送首部块字节序列 |


- HTTP2 中帧 type 和 flags 可能的组合

![](/images/http2中帧type和flags可能的组合.png)

#### 5.3.5 几种常见帧的格式结构

##### Data帧

```
+---------------+
|Pad Length? (8)|
+---------------+-----------------------------------------------+
|                            Data (*)                         ...
+---------------------------------------------------------------+
|                           Padding (*)                       ...
+---------------------------------------------------------------+


DATA Frame {
  Length (24),
  Type (8) = 0x00,

  Unused Flags (4),
  PADDED Flag (1),
  Unused Flags (2),
  END_STREAM Flag (1),

  Reserved (1),
  Stream Identifier (31),

  [Pad Length (8)],
  Data (..),
  Padding (..2040),
}
```

这个data frame里字段， length，type， unused flags， reserved 和 stream identifier 在5.4.3小节有介绍。 下面介绍data frame里的额外字段：

- pad length(8): 一个8bit的字段，表示填充物padding的长度。 这个字段出现是有条件的，只有flags设置为padded情况下才会出现。
- Data： 传递的应用程序数据。 Data数据长度等于payload长度减去其他出现字段的长度。
- padding: 填充的字段，没有具体语义，发送时必须设为0，作用是混淆报文长度。

DATA帧的标识有:

- END_STREAM(0x01): bit的位0设置为1，代表当前流的最后一帧
- PADDED(0x08): bit的位3设置为1代表存在padding

##### HEADERS帧

```
+---------------+
|Pad Length? (8)|
+-+-------------+-----------------------------------------------+
|E|                 Stream Dependency? (31)                     |
+-+-------------+-----------------------------------------------+
|  Weight? (8)  |
+-+-------------+-----------------------------------------------+
|                   Header Block Fragment (*)                 ...
+---------------------------------------------------------------+
|                           Padding (*)                       ...
+---------------------------------------------------------------+


HEADERS Frame {
  Length (24),
  Type (8) = 0x01,

  Unused Flags (2),
  PRIORITY Flag (1),
  Unused Flag (1),
  PADDED Flag (1),
  END_HEADERS Flag (1),
  Unused Flag (1),
  END_STREAM Flag (1),

  Reserved (1),
  Stream Identifier (31),

  [Pad Length (8)],
  [Exclusive (1)],
  [Stream Dependency (31)],
  [Weight (8)],
  Field Block Fragment (..),
  Padding (..2040),
}
```

主要看HEADERS frame payload这部分字段

- pad length(8): 8 bit长度，指定下面字段padding长度。 该字段只有在padded标识被设置后才会出现。
- exclusive（1）： 一个bit标识，这个字段只有在priority标识被设置时才会出现。
- stream dependency(31): 指定一个长度为31 bit的stream identifier。 该字段只有在设置了priority标识时才会出现。
- weight（8）： 一个无符号的8bit证书。 该字段只有在设置了priority标识时才会出现。
- field block fragment： header块片段
- padding： 填充的八位字节，没有语义，发送时设置填充值为0

headers帧的标识有：

- END_STREAM(0x01): bit的位0设置为1，标识当前header块发送的最后一块，但是带有end_stream标识的headers帧后面还可以跟continuation帧
- end_headers(0x04): bit 的位2设置为1，表示此帧包含整个字段块，并且后面没有continuation帧，没有设置end_headers表示的headers帧必须跟在同一流的continuation帧之后。
- padded（0x08）： bit的位3设置为1，PADDED 设置后表示 Pad Length 字段以及它描述的 Padding 是存在的。
- priority(0x20): bit 的位5设置为1，表示存在 Exclusive Flag (E), Stream Dependency 和 Weight 3 个字段。


## 六 http2重要特性详解

### 6.1 多路复用

在http1.1中，一个http的数据传输需要建立一个tcp连接，虽然有pipeline特定，但是又有头部阻塞的问题。

在http2中，在一个tcp连接中，可以发起多个http2连接请求，而每个http2连接中又可以发起多个流来传输数据。这都得益于http2中数据格式的设计，最重要的流概念。

HTTP1.1 和 HTTP2 的连接传输对比图

![](/images/http2连接传输对比图.png)


在 HTTP2 中用这个 stream ID 来标识帧和流的对应关系。


### 6.2 头部压缩

HTTP 1.1 请求头的协议内容很多，而且大部分都是重复的。在 HTTP1.1 中每次请求都会大量携带这种冗余的头信息，浪费流量。

在 HTTP2 中，设计了 HPACK 压缩算法对头部协议内容进行压缩传输，这样不仅数据传输速度加快，也能节省网络流量。

HPACK 原理:

客户端和服务端共同维护了一份静态字典表（Static Table），其中包含了常见头部名及常见头部名称与值的组合的代码。
客户端和服务端根据先入先出的原则，共同维护了一份能动态添加内容的动态字典表（Dynamic Table）。
客户端和服务端支持基于静态哈夫曼码表的哈夫曼编码（Huffman Coding）


### 6.3 服务端推送

服务端推送是一种在客户端请求之前发送数据的机制。在 HTTP2 中，服务器可以对客户端的一个请求发送多个响应。除了对原始请求响应外，还可以向客户端推送额外的数据。

服务端推送的目的是让服务器通过预测它收到请求后有哪些相关资源需要返回，从而减少资源请求往返次数。

比如在 HTML 页面的请求后，通常是对该页面应用的样式表和脚本的请求，当这些资源被服务端直接推送给客户端时，客户端就不需要单独给服务器发送请求来获取这些资源了。

![](/images/http2服务端推送.png)

在 page.html 文件中包含资源文件 script.js 和 style.css。客户端向服务端请求 page.html 文件，服务端发现 page.html 文件中包含了这两种资源文件，就会把这两种资源文件推送给客户端，以此来减少客户端的请求次数。如上图。

所有服务端推送数据流都由 PUSH_PROMISE 帧发起。

>在实践中，服务端推送很难有效使用。因为需要服务端正确预测客户端发出的额外请求，预测必须同时考虑缓存、内容协商和用户行为等因素。预测错误又可能导致性能下降，因为服务端发送了额外的数据。特别是推送了大量数据时可能与重要的响应数据发生线路争用等问题。
>客户端可以请求禁用服务端推送，SETTINGS_ENABLE_PUSH 设置为 0 就可以了。

### 6.4 流量控制

在 HTTP2 中，使用流来实现多路复用，这种会对 TCP 连接使用产生竞争，从而导致流传输被阻塞。流量控制是确保在同一连接上的流不会相互干扰。流量控制既能用于单个流也能用于整个连接。

HTTP2 使用 WINDOW_UPDATE 帧提供流量控制功能。

#### 6.4.1 流控的一些原则

HTTP2 流控协议旨在不修改协议基础上使用各种流控算法，流量控控的一些特点：


1. 流量控制作用于连接。所有类型流控都是作用于两个单跳的端点之间，而不是整个端到端的链路
2. 流量控制基于 WINDOW_UPDATE 帧实现。接收端公布自己打算在每个流及整个连接上接收多少个八位字节。这是一个基于信用的方案。
3. 流量控制具有方向性，总体控制是由接收端提供的。接收端可以为每个流和整个连接设置它所需要的任何窗口大小。发送端必需遵循接收端设置的流量控制限制。客户端、服务端和中间设施，作为接收端均独立声明其流控窗口，并在发送时同样遵循其对等端设置的流控限制。
4. 对于所有的流和连接，流量控制窗口的初始值大小为 65535 字节。
5. 帧类型也决定了是否要在这个帧上应用流控。对于在 rfc9113 文档中提到的帧类型，只有 DATA 帧是流控的作用对象。其他所有帧均不占用流控窗口空间，这确保了重要的控制帧不会被流控阻塞。
6. 端点可以选择禁用自己的流量控制，但端点不能忽略来自其对等端点的流量控制信号。
7. HTTP2 中仅定义了 WINDOW_UPDATE 帧的格式和语义。在这篇文档 rfc9113 中没有规定接收端如何决定何时发送此帧或发送的值，也没有规定发送端如何选择发送数据包。协议的实现者可以选择任意一种符合其需求的算法。


### 6.5 流优先级

在 HTTP2 中，一个消息可以拆分为多个单独的帧，并且允许来自多个流的帧被多路复用，客户端和服务端帧传输可能是乱序传输，所以优先级顺序就变成了一个关键性能考虑因素。还有一种重要情况是，当发送受到限制情况下，可以通过优先级顺序选择那个流传输帧。显示设置过优先级的流将被优先安排。但这种优先并不能保证一个优先级高的流能得到优先级处理或优先级传输。所以说，优先级仅仅作为一种建议存在。

HTTP2 允许每个流具有关联的权重和依赖性：

- 每个流可以分配一个 1-256 范围之间整数权重
- 每个流都可以被赋予对另外一个流的依赖性

