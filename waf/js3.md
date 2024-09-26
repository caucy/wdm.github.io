# ja3 指纹

## 什么是ja3?
ja3 是区分客户端tls的一种技术。tls 握手的时候，一些信息可以给客户端生成指纹，发生网络攻击的时候，可以拿到观察指纹特征，找出可能发生攻击的客户端。

ja3 的格式是
```
ssl_version,cipers,extensions
```

其中ciper 是10进制数

抓包举例：
```

```

## ja3的应用
通过ja3 可以做ddos 安全防护


## 如果避开ja3检测
控制ciper 的生成，可以避开ja3 特征检测

## nginx 如何生成ja3
nginx 生成ja3，可以参考https://github.com/fooinha/nginx-ssl-ja3/blob/master/patches/nginx.1.23.1.ssl.extensions.patch， 不过patch 实现，ssl version 有点问题
