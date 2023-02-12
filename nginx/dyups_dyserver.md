# nginx的痛点

## nginx reload 的危害
在重新加载配置文件时，nginx 会退出老worker，启动新work，往往会导致系统内存，cpu，耗时等资源的上升，很容易因为reload 导致业务有损，所以在真实业务中，我们往往会尽量少的reload Nginx 配置以确保服务的稳定性和可用性。

## 一个典型的nginx 配置

非常常见的一个nginx 例子如下：
```
http {          
       upstream  myserver{        
            #定义upstream名字，下面会引用         
            server 192.168.1.100;        
            #指定后端服务器地址         
            server 192.168.1.110;        
            #指定后端服务器地址         
            server 192.168.1.120;        
            #指定后端服务器地址     }     

       server {         
          listen 80;         
          server name www.myserver.com;         
          location / {             
              proxy_pass http://myserver;        #引用upstream         
          }     
       } 
     }

```

上面的使用方法有两个痛点：
* upstream 变更会导致realod
upstream 往往要走公司的服务发现，server ip 会动态变更，如果不是提供的vip 的话，一旦ip 漂移，会导致流量502.但在云原生环境中，发生ip漂移是非常常见的。

* server 变更会导致reload
添加host， 变更location 是非常常见的需求，如何减少reload，也是非常重要的。


# 1. nginx 如何实现不reload 更新upstream

开源库[http_dyups](https://tengine.taobao.org/document/http_dyups.html) 提供了一个可插拔的module 实现通过api curl nginx 就能变更upstream 的功能，不会reload nginx。 


# 2. nginx 如何实现不reload 更新server


# 3. apisix 如何实现不reload 更新upstream & server
