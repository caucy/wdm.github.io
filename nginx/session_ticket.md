# nginx tls 握手的ticket rotate 细节

## 1. tls 握手的流程
一般完整握手流程是9步骤，非常慢

## 2. tls 握手session复用
session 存在后，可以加快tls 握手，最快速度拿到对称秘钥。

session 握手的流程：

1. 第一次握手后,服务器会生成一个session id,这个session id 会传给'浏览器'
 
2. 当在'一定'时间之内,浏览器或'客户端'携带这个'session id'访问服务器
 
3. 服务器从'自己的内存'或'其它缓存中'获取到这个'session id'对应的'加密密钥'
 
4. 服务端给'客户端'说就使用'上一次'的'加密密钥'就可以,不需要使用'DH|ECDH'再次生成新的密钥
 
备注： 
 
  1、对'client'的要求: 在一定的时间内缓存'session id'以及自身所对应的'加密密钥'
 
  2、对'server'的要求：在一定的时间内缓存'session id'以及自身所对应的'加密密钥'
  
缺点：
  1、server 缓存session id 对应的数据，需要一个远程调用存到共享的memcache，或者redis 去

  

## 3. tls 握手ticket方案

基于ticket 的方案：

1. server端存放session信息 'session id'和'密钥'
 
2. 而是基于一个'独有'的密码,这个密码是'server 集群'共同分享的密码
 
3. 基于这个密码把'相关的信息'加密,发送给'client'
 
4. 客户端'下一次'需要握手的时候,把相关的'加密信息'发送给'server端'
 
5. 只有这个'server 集群'内的机器才知道这个'密码',然后解密,获取上一次握手成功后的'密钥'
 
6. 基于此,server端就可以与'基于内存中保存的密钥的client'进行通信

优点：
  1. 大幅节约内存
     
  3. server 缓存了ticket_key，只需要定时更新即可。


## 4. ticket_key 复用方案
nginx 实现一个定时器，定时去keyserver 去取ticket_key 即可，定期发送一个subrequest。

具体更新ticket_key 可以参考lua [ngx_http_lua_ffi_update_ticket_encryption_key](https://github.com/openresty/lua-ssl-nginx-module/blob/master/src/ngx_http_lua_ssl_module.c#L189C1-L189C46) 的实现,核心就是更新ssl_ctx 中的ngx_ssl_session_ticket_key_t


参考文档：
1. https://blog.csdn.net/wzj_110/article/details/133241634
