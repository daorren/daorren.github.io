---
title: 浅谈不同版本HTTP协议
date: 2017-02-08 20:07:00
tags: HTTP
---
做Web开发的同学，肯定对互联网的基石——HTTP协议有所了解吧，据说这也是技术面试的时候经常出现的问题呢！

这里我从比较协议版本异同的角度，带大家回顾一下经典的知识。

<!-- more -->

## HTTP的历史
最初版本`HTTP/0.9`，没有请求头，HTTP方法只有`GET`，服务器响应的必须是HTML文档。

到了`HTTP/1.0`，加入了请求头和更多请求方法，并且响应内容不再局限于超文本。

`HTTP/1.1`是现在部署最广泛的版本，`rfc7230`-`rfc7235`在`rfc2616`的基础上，对`HTTP/1.1`做了大量修订。

但由于协议包含了太多细节和可选的部分，体积过于庞大，迄今为止没有浏览器实现了所有细节。

最新版本的`HTTP/2`起源于`SPDY`，后者是谷歌牵头开发的协议。`HTTP/2`是二进制协议，删除或减少可选的部分，并且将不再使用小版本号。

尽管HTTP/2 的 TLS（会话层） 是可选选项，但所有浏览器都不支持 `HTTP/2 ClearText`，只支持 `HTTP/2 Over TLS`，所以在浏览器端，`HTTP/2 `是需要基于 `HTTPS` 部署的。

## HTTP/1.1新增的主要功能
### 持久连接（Persistent Connections）
也就是 `Connection:keep-alive`，服务器返回请求之后不会立刻关闭连接，允许客户端使用同一个连接发送后续HTTP请求。这在`HTTP/1.1`是默认开启的，除非指定 `Connection: close` 首部。

好处是，使网页响应更快。但是对于访问量大的服务器来说，他要维持更多连接会有更大的开销，所以会关掉，比如大型电商网站。

### HTTP 管道（Pipelining）
客户端把多个HTTP请求放到一个TCP连接中一一发送，而在发送过程中不需要等待服务器对前一个请求的响应。

但是服务器是要按照顺序处理请求的，如果前一个请求非常耗时，后续的请求都会受影响，在客户端这里导致一个队首阻塞的问题，所以现代浏览器都关闭这个功能。

### Chunked 编码（Chunked Encoding）
首先我们来看看，`Content-Length`这个有趣的响应头，它的作用是告诉客户端响应的大小。然而在`HTTP/1.1`，这是个可选的，因为对于动态生成的内容，事先不知道内容大小。

然而在 `persistent connections` 和 `pipelining` 的状况下，如果在同一个连接的多个请求中，中间某个响应没有 `Content-Length` 头，客户端会把后续请求的响应，一起当做这个请求的响应。

而`chunked encoding` 的出现就是为了解决这个问题。当服务器生成动态内容的时候，它会带上 `Transfer-Encoding: chunked` 头，并以多块已知大小的小数据形式返回响应，最后一个chunked块的大小为0，这样客户端就知道一个响应结束了。而客户端这块，不需要等到内容字节全部下载完成，只要接收到一个chunked块就可解析页面。

### Etags缓存机制（Etags Based Cache）
浏览器发起一个请求，后端返回响应里面，包含 `Last-Modified/ETag`。

浏览器再次请求时，首部带上 `If-None-Matched`（存储的是Etag值），后端再次生成etag值并与该值比对，如果两个值相同，生成的内容不返回，只返回 `Status: 304 Not Modified`，客户端使用本地缓存。

对于Rails，其生成机制类似于 `headers['ETag'] = Digest::MD5.hexdigest(body)`。

所以实际上，如果使用默认的 Rack::ETag 中间件，后端的代码template其实都执行了一遍。

所以推荐做法是，在Controller中，手动指定 etag。
```ruby
def show
  @article = Article.find(params[:id])
  fresh_when :last_modified => @article.updated_at.utc, :etag => @article
end
```
额外说一下浏览器的缓存机制。使用 `Cache-Control` 首部控制
- `no-cache` 表示先与服务器确认响应是否更改，如果存在Etag且验证资源未被更改，可以避免下载。`no-store` 则禁止浏览器和中继缓存的响应，每次都会下载完整响应。
- `private` 表示只为单个用户缓存，不允许中继缓存进行缓存。
- `max-age` 指定从当前请求开始，允许获取的响应被重用的最长时间（单位秒）。

### 内容协商（Content Negotiation）
同一资源，服务器上可能有多个版本，那么就需要客户端告知服务器自己的偏好，服务器根据偏好选择合适的版本进行响应。

- 客户端驱动
	客户端发起请求，服务器发送可选项列表，客户端作出选择后在发送第二次请求。
- 服务器驱动（应用广泛）
	服务器检查客户端的请求首部集并决定（猜测）提供哪个版本。

请求头主要是Accept首部集（`Accept`、`Accept-Language`、`Accept-Encoding`等），响应头主要是`vary`首部。

`Accept-Language: en;q=0.5, fr;q=0.0, nl;q=1.0, tr;q=0.0`，表示用户最愿意接受荷兰语（nl），英文也行（en）,就是不愿意接受法语（fr）或者土耳其语(tr)。q质量值表示客户端的偏好，范围从0.0~1.0（1.0优先级最高）

`Vary`首部主要是web服务器告知缓存服务器，根据其内容来选择文档或产生定制内容。举个例子，web服务器添加响应首部`Vary: Accept-Encoding`，告知代理服务器根据客户端的请求首部Accept-Encoding缓存不同的版本，这样下次客户端请求同一资源时，根据Accept-Encoding选择相应的缓存版本响应。

### 协议升级（Protocol Switching）
HTTP/1.1 引入了 Upgrade 机制，它使得客户端和服务端之间可以借助已有的 HTTP 语法升级到其它协议。

要发起协议升级，客户端必须在请求首部中指定这两个字段：
```
Connection: Upgrade
Upgrade: protocol-name[/protocol-version]
```
如果服务端不同意升级或者不支持 Upgrade 所列出的协议，直接忽略即可（当成 HTTP/1.1 请求，以 HTTP/1.1 响应）；如果服务端同意升级，那么需要这样响应：
```
HTTP/1.1 101 Switching Protocols
Connection: upgrade
Upgrade: protocol-name[/protocol-version]
```
HTTP `Upgrade` 响应的状态码是 101，并且响应正文可以使用新协议定义的数据格式。

这个机制常用来建立WebSocket连接，也可以用来升级到HTTP/2。

### Host 首部（Host Header）
使用`HTTP/1.1`发送的请求，必须指明host。

因为在服务器只知道请求的IP，如果两个域名指向同一个IP，`HTTP/1.0`是无法区分的。但是携带host，就能区分了。
### Ranges 首部（Rangers Header）
允许浏览器请求某个文档的一部分。比如，请求一个文档的第二个500 byte，只需包含 `Range: bytes=500-999` 头。

功能：下载断点续传

### 消息完整性检查（Message Integrity Checks）
原本上有个 `Content-MD5` 头，用作完整性检查，但是后来被反对了。这个应该是新标准[rfc3230: Instance Digests in HTTP](HTTPs://www.ietf.org/rfc/rfc3230.txt)。

### 摘要认证（Digest Authentication）
摘要认证，是一种用户认证方法。相比于基本认证，它通过一些手段增加了安全性。

客户端请求某个需要认证的URL，服务端返回401未认证状态，并且在返回的 `WWW-Authenticate` 头中包含了 `Digest`（digest这种认证方式），`realm`（领域），`QOP`(保护质量，“auth”表示只进行身份查验)，`nonce`（一串随机值，在下面的请求中会一直使用到，当过了存活期后服务端将刷新生成一个新的nonce值），`algorithm`（摘要算法）。

不同情况下算得response值 [参考wiki](HTTPs://www.wikiwand.com/en/Digest_access_authentication)，把值写入 `Authorization` 头，发给服务器。

## HTTP/2 新增功能概述
`HTTP/2` 并没有改动 `HTTP/1` 的语义部分，例如请求方法、响应状态码、URI 以及首部字段等核心概念依旧存在。`HTTP/2` 最大的变化是重新定义了格式化和传输数据的方式，这是通过在高层 HTTP API 和低层 TCP 连接之间引入`二进制分帧层`来实现的。这样带来的好处是原来的 WEB 应用完全不用修改，就能享受到协议升级带来的收益。

### 核心概念
- 连接（Connection）：与 `HTTP/1` 相同，都是指对应的 `TCP` 连接
- 流（Stream）：存在于连接中的一个虚拟通道。流可以承载双向消息，每个流都有一个唯一的整数 ID（一个请求响应的事务，在同一个流中。）
- 消息（Message）：一条消息，指的是逻辑上的 HTTP 消息，比如一个请求或者一个响应等，消息由一个或多个帧组成
- 帧（Frame）：`HTTP/2` 数据通信的最小单位。帧用来承载特定类型的数据，如 HTTP 首部、负荷；或者用来实现特定功能，例如打开、关闭流。每个帧都包含帧首部，其中会标识出当前帧所属的流

### 主要改变
#### 二进制协议（Binary Protocol）
二进制协议解析起来更快，更重要的是，相比文本协议，它更少出错。
#### 单一TCP（One TCP Connection Per Origin）
`TCP` 协议本身更适合用来长时间传输大数据，这样它的稳定和可靠性才能显露出来。`HTTP/1` 时代太多短而小的 `TCP` 连接，反而更多地将 `TCP` 的缺点给暴露出来了。
当然，这里指的是一个源只要一个连接，多个源的话，还是要多个连接。[阮一峰JS教程：同源政策](HTTP://javascript.ruanyifeng.com/bom/same-origin.html)）
#### 多路复用（Multiplexed）
一个连接，能同时打开多个流。也就是说，多个HTTP请求和响应能同时在同一个连接中发生。（既然能这么做了，当然一个TCP连接也够了）
#### 拥有优先级（Prioritized）
同时能进行多个请求和响应，所以需要控制顺序。`HTTP/2` 允许给每个流附上 `weight`（1~256）和对其他流的依赖，这最终会构建一个优先级树。
#### 流量控制（Flow Control）
对于每个流来说，两端都必须告诉对方自己还有更多的空间来接受新的数据，而在该窗口被扩大前，另一端只被允许发送这么多数据。
#### 服务端推送（Server Push）
允许服务器在给原本请求返回响应的同时，还能主动返回其他资源给客户端。

在 `HTTP/1.X` 的时代，有很多连接数优化的网站性能优化方式，比如资源内联。其实到了新版本，开启服务端推送，就能帮你解决问题，提升性能。
#### 首部压缩（Header Compression，HPACK）
HTTP本身是无状态的，所以请求头通常携带不少信息。当一个客户端从同一服务器请求大量资源的时候，请求头看起来几乎都是一致的，它们值得压缩。
SPDY协议原本有首部压缩，但是有安全问题，所以 `HTTP/2` 专门设计了 `HPACK`。
它是在服务器和客户端各维护一个“首部表”，表中用索引代表首部名，或者首部键-值对，上一次发送两端都会记住已发送过哪些首部，下一次发送只需要传输差异的数据，相同的数据直接用索引表示即可，另外还可以选择地对首部值压缩后再传输。


参考资料
[1] [JerryQu 的小站：HTTP专题](HTTPs://imququ.com/series.html)  （**强烈推荐**）
[2] [Apache Week: HTTP/1.1](HTTP://www.apacheweek.com/features/HTTP11)
[3] [Gitbook: HTTP2](HTTPs://bagder.gitbooks.io/HTTP2-explained/zh)
[4] [Rails 3 和 Rails 4 中 ETags 工作原理](HTTPs://ruby-china.org/topics/24996)
[5] [Google: HTTP 缓存](HTTPs://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/HTTP-caching)
[6] [Why is pipelining disabled in modern browsers?](HTTP://stackoverflow.com/questions/30477476/why-is-pipelining-disabled-in-modern-browsers)
[7] [HTTP/2 与 WEB 性能优化（二）](HTTPs://imququ.com/post/HTTP2-and-wpo-2.html)
[8] [Google: Introduction to HTTP/2](HTTPs://developers.google.com/web/fundamentals/performance/HTTP2)
