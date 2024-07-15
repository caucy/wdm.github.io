# 4 7 层如果做ip传递的

## 背景
流量经过lvs 后，如何将vip 传递后rs 这是一个比较实在的问题，在一些业务逻辑里，rs可能是需要vip的。目前方案主要有proxy_protocal, toa, xff
一个主要的实现是pp 协议，一个是通过tcp option, 这两种方式都可以实现。xff 一般可能被覆盖，不是很可靠，很少用。

## proxy_protocal 的实现
一个简单的demo 就可以看到proxy_protocal v1 的原理, 如果80 nginx 开了pp的话，可以用curl 直接访问
```
curl 127.0.0.1 -d '1111' --haproxy-protocol -v

PROXY TCP4 127.0.0.1 127.0.0.1 54208 80
> POST / HTTP/1.1
> User-Agent: curl/7.64.0
> Accept: */*

```
proxy_proxy 的原理就是三次握手后，在post 数据前加个前缀，前缀包括了协议版本，源ip port，目标ip port.

v1 和v2 稍微有点不同，头部换成二进制，加了更多扩展属性。现在普遍都用v2。
ipv4:
![image](https://github.com/caucy/wdm.github.io/assets/19687952/e68073b0-5158-4205-82d7-0a573bc059c1)

ipv6:

![image](https://github.com/caucy/wdm.github.io/assets/19687952/011b76d9-e954-426d-a710-882903339e57)


## nginx 使用proxy_protocal
nginx 开启proxy_protocal 有两个层面，一个层面是nginx.conf 中的配置，一个是自己模块开发使用。
### 1. 配置开启pp

* 监听端口开启pp
一般listen 后加proxy_protocal 就可以
```
http {
    #...
    server {
        listen 80   proxy_protocol;
        listen 443  ssl proxy_protocol;
        #...
    }
}
```

* 注意，如果加了proxy_protocal，那所有请求都要走pp协议，有些厂商会在这兼容，可能需要定制化改造下nginx 源码。*
### 2. 源码定制开发使用pp
可以参考proxy_protocol_server_addr 的获取，proxy_protocol 的地址都会存在connection->proxy_protocol属性中，可以直接使用。但是需要关注v1还是v2.


## toa 的实现
TOA 名字全称是 tcp option address，是 FullNat 模式下能够让后端服务器获取 ClientIP 的一种实现方式，它的基本原理比较简单。

客户端用户请求数据包到达 LVS 时，LVS 在数据包的 tcp option 中插入 src ip 和 src port 信息。

数据包到达后端服务器（装有 toa 模块）后，应用程序正常调用 getpeername 系统函数来获取连接的源端 IP 地址。

由于在 toa 代码中 hook（修改）了 inet_getname 函数（getpeername 系统调用对应的内核处理函数），该函数会从 tcp option 中获取 lvs 填充的 src 信息。

这样后端服务器应用程序就获取到了真实客户端的 ClientIP，而且对应用程序来说是透明的。

![image](https://github.com/caucy/wdm.github.io/assets/19687952/33613058-85e6-48fd-8117-dfdd14d225d7)


### 方案优缺点
toa 方案有个比较明显的缺点，tcp option 能被覆盖或者内核模块可能重启等各种原因没启用，选用pp协议更安全。
