---
title: 如何让外网访问本机服务
date: 2017-11-18 10:41:43
tags: 
- frp
- ngrok
---
大家平时在工作中，可能会遇到这样的问题：本机处在一个内网环境中、没有外网IP，但如果希望运行在本机的服务，让外网也能够访问，怎么做到呢？

接下来就介绍几种方法，可以解决这样的问题，思路都是一致的，依靠一个拥有公网IP的机器中转，把内网映射出去。

## 简单法：直接运行 ngrok/localtunnel
[ngrok](https://ngrok.com/download) 和 [localtunnel](https://github.com/localtunnel/localtunnel) 都是开箱即用的工具，其原理是服务提供商提供了域名，为你的服务做转发。你不需要自己准备中转的机器，只要下载这俩工具到本机，对于临时的访问需求可以直接使用。

但是这两个工具又各有缺点，ngrok 首先是被墙了，其次如果你不是付费用户，每次启动 URL 都会发生变化；而 localtunnel 经常异常崩溃，可能需要额外的重启脚本。

也就是说，如果你的情况比较特殊，比如需要和外部服务对接，映射出去的你本机的服务需要越稳定越好，这两种方式都不会是首选。

<!-- more -->

## 普适：frp
[frp](https://github.com/fatedier/frp) 是一个服务端-客户端架构的软件，其原理是你在作为中转的机器上安装 frp 服务端（称为 frps），在你的本机安装 frp 客户端（称为 frpc）。根据各自的配置，frps 会监听访问某个端口的请求，转发给与其已建立好连接的 frpc，frpc 再把请求转给你本机实际运行的服务，收到请求后整个转发链条再反过来。

下面贴一个配置示例，其原理类似 ssh 隧道转发 tcp。（实际 frp 还支持其他，有兴趣的同学可以去探索一下）
```shell
# frpc.ini
[common]
server_addr = <server ip>
server_port = <frp port>  # frpc connect frps

[exposed]                 # just a name
type = tcp
local_port = 3000         # local service port
remote_port = 10003       # for use


# frps.ini
[common]
bind_port = 7000          # open for frpc
```

准备好一个带有公网IP的机器，下载好 frp ，做好 frps 的配置，然后执行 `frps -c frps.ini`；然后在你的本机也下载好 frp，如上进行 frpc 的配置（其中 `local_port` 是本机服务的运行端口，`remote_port` 是最终访问的端口），然后启动 `frpc -c frpc.ini`。

接着访问 `<server ip>:<remote_port>` 时，请求会被转发到你本机的 `localhost:<local_port>`，完成！

## 更多：自建 ngrok
尽管 frp 已经是个自建的转发服务，你拥有相当的控制权了，有时候还是不那么令人满意，比如我的需求是转发 HTTPS 服务（HTTPS 需要域名，光是公网 IP 不够）。如果还是使用 frp，尽管其内建支持 HTTPS，配置里却没有证书路径这一项；如果是和 Nginx 配合（安装 LetsEncrypt 颁发的 HTTPS 证书时，直接走的 Nginx 集成），也一直不能成功。

接着我 Google 到 [搭建 ngrok 服务实现内网穿透](https://imququ.com/post/self-hosted-ngrokd.html)，自建 ngrok 的工作原理跟第一种方式提到的 ngrok 差不多，只不过你自己就是域名提供者。在引用的这篇博文中，具体做法说得非常清楚，我不再赘述。


但是结合 [Ngrok self-hosting](https://github.com/inconshreveable/ngrok/blob/master/docs/SELFHOSTING.md) 文档，我要做一些说明：

- 编译服务端和客户端，除了环境变量和 Go 的包，其他都不依赖，尤其是不依赖作者在博文中生成的证书。基于此，任何编译出来的二进制文件都是不依赖域名或者证书的，我把 linux 平台和 Mac 平台的编译好的文件放在 [release](https://github.com/daorren/ngrok/releases) 中，可以直接使用。
- 作者会基于域名生成证书，前提是假设没有域名的 HTTPS 证书。如果你已经购买，或者从 [letsencrypt](https://letsencrypt.org/) 获取了证书，也就没有必要做这一步。
- 对于 HTTPS 绑定子域名的情况，比如 `pub.example.pub`，需要在客户端启动时指定，否则将是随机生成的一个 subdomain。
  ```shell
  # server
  sudo ./bin/ngrokd -tlsKey=/<path to>/pub.example.pub/privkey.pem -tlsCrt=/<path to>/live/pub.example.pub/fullchain.pem -domain="machen.pub" -httpAddr=":8082" -httpsAddr=":8083"

  # client
  ./ngrok -subdomain pub -proto=https -config=ngrok.cfg 3000
  ```

总结：如果需求特简单、自己也没有主机，直接下载 ngrok/localtunnel 使用即可；如果要求服务稳定 URL 不变化，且有拥有外网IP的主机，可以使用 frp，一次配置以后都可以用；如果有需求用 HTTPS 的，可以自建 ngrok。





