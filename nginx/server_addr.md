# 4 7 层如果做ip传递的

## 背景
流量经过lvs 后，如何将vip 传递后rs 这是一个比较实在的问题，在一些业务逻辑里，rs可能是需要vip的。目前方案主要有proxy_protocal 和 toa,
一个主要的实现是pp 协议，一个是通过tcp option, 这两种方式都可以实现。

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

## nginx 获取tcp option

