---
title: Http基础
date: 2018-08-14
tags:
- OkHttp
categories:
- 框架源码
---

<!-- toc -->

#### WebSocket

WebSocket协议是基于TCP的一种新的网络协议。它实现了浏览器与服务器全双工(full-duplex)通信——允
许服务器主动发送信息给客户端。


在WebSocket协议协议产生之前，双工通讯是通过客户端向服务端不停的发送HTTP请求，从服务端拉取数据
更新来实现的。这样做造成了客户端的HTTP轮询的滥用，并且效率是底下的。

HTTP轮询导致的一系列问题：

1.服务器被迫为每个客户端创建许多不同的底层TCP连接：一个用于向客户端发送信息，其它用于接收每个传
入消息。

2.有些协议有很高的开销，每一个客户端和服务器之间都有HTTP头。

3.客户端脚本被迫维护从传出连接到传入连接的映射来追踪回复。

HTTP作为传输层的双向通讯技术,只能牺牲效率和可依赖性其中一方来提高另一方，因为HTTP最初的目的不是
为了双向通讯，WebSocket协议的出现，解决了HTTP的这些问题。

WebSocket协议提供一个更简单的解决方案——使用单个TCP连接双向通信。结合WebSocket API ，
WebSocket协议提供了一个用来替代HTTP轮询，实现网页到远程主机的双向通信的方法。

在实现websocket连线过程中，需要通过浏览器发出websocket连线请求，然后服务器发出回应，这个过程
通常称为“握手” 。在 WebSocket API中，浏览器和服务器只需要做一个握手的动作，然后，浏览器和服务
器之间就形成了一条快速通道。两者之间就直接可以数据互相传送。

#### 路由(route)

路由是指路由器从一个接口上收到数据包，根据数据包的目的地址进行定向并转发到另一个接口的过程。

在web开发中，“route”是指根据url， 分配到对应的Web 服务器处理函数。

路由就是一个路径的解析，根据客户端提交的路径，将请求解析到相应的控制器上；
从 URL 找到处理这个 URL 的类和函数。

一个url可以分为scheme、主机和具体路径，而路由就是根据具体路径，在Http Server中找到对应的处理函数。

个人理解，在ok这里的路由可以理解为连接服务器的一种途径，这个途径包含了代理的信息(可能没有)和特定
要连接的服务端目标地址(inetSocketAddress)；

一个url会因为代理服务器和DNS服务器协议的解析，而对应有多个IP地址和端口号，而Route就是根据这些协议，
去尝试连接对应的ip和端口，如果连接成功，则是有效的Route，否则是无效的；

可以这样认为，我们请求服务器的Address是不变的，每次选择出一个代理(Proxy),根据这个Proxy去解析
不同的协议，就可以得到多对(IP+端口)(DNS能够解析出多对)，每一对就代表一个inetSocketAddress，
而每个Route就是Address+Proxy+inetSocketAddress，也就是说，一个Proxy能够解析出多个Route；

Route的数量 = 每个Proxy解析出来的inetSocketAddress数量的和；

#### Cookie

Cookie是保存在客户端的轻量级的持久化方式，由服务端设置，用来保存客户端非隐私安全的信息；

Cookie在response时，由服务端进行设置，包含在响应头中，客户端读取到保存cookie的响应头可以决定
是否接受设置cookie的请求。如果客户端接受了cookie，则在request时，会读取出cookie信息，通过请
求头传递给服务端；

服务端设置cookie的格式为：
```
Set－Cookie: NAME=VALUE；Expires=DATE；Path=PATH；Domain=DOMAIN_NAME；SECURE
```
其中的NAME=VALUE键值对就是实际要保存的cookie信息，必须放在cookie信息的最前，其他的键值对则是
cookie自带的属性，用来描述该cookie的信息；

客户端对服务端发起请求时，会将cookie添加到请求头中，格式为：
```
cookie:NAME=VALUE；
```
只会携带之前服务端设置的NAME=VALUE键值对，当然我们也可以在cookie中额外添加其他的信息，用来作为
传递给服务端的数据；


#### 客户端缓存

客户端缓存大致可以分为两种情况，一种是强制缓存，另一个种是对比缓存；

强制缓存：当缓存生效时，则不再请求服务端数据；
![image](/images/强制缓存.png)


对比缓存：通过请求服务端来确认缓存是否有效；
![image](/images/对比缓存.png)

客户端使用缓存的流程：
![image](/images/客户端重用缓存流程.png)

##### 缓存头数据

客户端缓存的缓存策略包含在响应头中，包括：

**expires**

服务端指定缓存到期时间，由于服务端和客户端时间可能不同步，所以在HTTP1.1之后不适用；

**Cache-Control**

```
Cache-Control：public      表示缓存是共有的，客户端和服务端(包括代理服务器)都可以缓存；

Cache-Control：private     表示缓存只能在客户端中被缓存(代理服务器不能缓存)；

Cache-Control：no-cache    如果是出现在响应头中，则表示客户端在使用缓存之前，需要再次发送一个
请求与服务端进行response验证(即是对比缓存的情况)；如果出现在请求头中，则表示本次请求不使用缓存，
直接进行网络请求来获取响应；

Cache-Control：no-store    表示不缓存任何客户端请求和服务端响应；

Cache-Control：max-age=60  表示缓存的有效时长(60秒)

Cache-Control：must-revalidate  缓存在使用之前，必须验证旧缓存的状态，一旦发现是过期的，则
禁止使用；(应该是与max-stale互斥的)

Cache-Control：max-stale  如果缓存是过期的，则最多过期多少时间，超过这个时间缓存将不能再被
使用；

Cache-Control：min-fresh  指定一个最小秒数，每次请求缓存都会重置减小这个时间，在这个最小时间
内，缓存始终是有效的(新鲜的)，超过这个时间，并且缓存过期了，则缓存失效；(作用类似于设定个担保时间，
在这个时间内，缓存肯定是能用的。)

Cache-Control：only-if-cached  只用在请求头中，表示如果已经有缓存了，则始终只使用缓存，否则
不存在缓存，则返回一个504响应状态码；(这个标识下，禁止走网络请求，所以在对比缓存下，拿不到缓存，
也会报504)

Cache-Control：no-transform    不允许对响应内容进行转换操作。代理服务器不能够对
Content-Encoding, Content-Range, Content-Type进行修改。例如，隐匿的代理服务器为了节省缓
存空间，可能会转换压缩原服务器发送的图片格式，而通过设置该标志，则这些行为是不被允许的。

Cache-Control：s-maxage       这个标识会覆盖max-age和Expires，但只适用于共享缓存(Public);      
```

上面的标识是可以多个组合使用的，之间用逗号隔开：
```
Cache-Control: no-cache, no-store, must-revalidate
```

**Pragma**

HTTP/1.0提供的通用头标记，可以用来实现多种效果(个人理解为可以用来代替多种头标记)；
一般是用来兼容HTTP/1.0协议，实现Cache-Control(HTTP/1.1才有)效果；

用法：

```
Pragma: no-cache

等价于

Cache-Control：no-cache
```
**Age**

如果为0，标识响应数据是直接从原始服务器中获得；
如果大于0，通常表示从原始服务器生成响应头的时间与代理服务器当前时间的时间差，即响应消息在代理服务器
存储的时间；

**Date**

原始服务器生成响应数据的时间；

**Last-Modified/If-Modified-Since**

客户端的对比缓存，就是通过这两个头数据来实现的；

- Last-Modified

该缓存标志携带在服务端返回的响应头中，记录了响应体在服务端最后一次的修改时间。客户端拿到之后，将
该标志和缓存一起保存起来，在下次请求服务端时，时间将被带上；

![image](/images/last_modified.png)

- If-Modified-Since

对应着Last-Modified，在之前缓存过服务端返回的数据后，再次请求服务端数据时，会将之前缓存的
Last-Modified标志的时间取出来，设置给If-Modified-Since，并通过请求头传递给服务端。

服务端在读取到请求头中If-Modified-Since标志时，会把服务端本地资源最后的修改时间与该标志的时间
进行对比，如果本地修改时间大于If-Modified-Since时间，说明自从上次请求之后，本地资源被修改过，
客户端的缓存过期，则响应整个内容，并返回200状态码。否则，说明客户端的缓存没有过期，可以继续使用，
并返回304状态码。

![image](/images/If-Modified-Since.png)


**ETag/If-None-Match**

> ETag/If-None-Match的优先级高于Last-Modified/If-Modified-Since

- ETag

是服务端返回的响应体的唯一标识(类似于生成MD5)，生成规则由服务端确定；

![image](/images/Etag.png)

- If-None-Match

当客户端之前缓存过服务端响应数据，并且保存了ETag标识，则再次请求服务端数据时，会把读取ETag标识
的内容，赋给If-None-Match，然后通过请求头传递给服务端。

当服务端从请求头中读取到If-None-Match标识后，会和本地的资源的标识做对比，如果不同，说明本地资源
修改过，重新生成了资源标识，相应的，客户端的缓存也就过期了，则响应整个内容，并返回200状态码。否则，
说明客户端的缓存没有过期，可以继续使用，并返回304状态码。

![image](/images/If-None-Match.png)


#### 连接与请求

**keepalive连接机制**

在传输数据后，仍然保持连接，处于空闲状态；当需要复用链接时，直接使用该连接，省去了三次握手、四次
挥手的环节，提高了请求速度；

由于连接管道的大小是确定的，如果同时存在太多的空闲连接，则会影响到客户端创建新的连接的链路速度；


**HTTP和HTTPS**
HTTP全称超文本传输协议，是应用最广泛的网络传输协议；

HTTPS全称安全套接字层超文本传输协议，可以认为是安全版的HTTP协议，因为HTTP协议是以明文进行数据发送的，而HTTPS是在HTTP基础上
增加了SSL层(对应SSL协议)，对传输的数据进行加密；

可以认为HTTPS = HTTP + SSL

**SSL和TLS**

SSL全称安全套接字层，位于TCP/IP协议与各种应用层协议之间，为数据通讯提供安全支持。可分为两层，分别
是SSL记录协议(SSL Record Protocol)，建立在可靠的传输协议之上(如TCP)，为高层协议提供数据封装、
压缩、加密等基本功能支持。SSL握手协议(SSL Handshake protocol)，建立在SSL记录协议之上，用于在
实际的数据传输开始前，通讯双方进行身份认证、协商加密算法和交换加密秘钥等。

TLS全称传输层安全协议，用于在两个通信应用程序之间提供保密性和数据完整性。同样包含了记录协议和握手协议。
TLS是基于SSL3.0协议规范之上的，可以认为是SSL3.1版本。

基本上可以认为SSL和TLS是对等的：

HTTPS = HTTP + TLS/SSL
