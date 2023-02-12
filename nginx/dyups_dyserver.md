# nginx的痛点

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
* 1. upstream 变更会导致realod
upstream 往往要走公司的服务发现，server ip 会动态变更，如果不是提供的vip 的话，一旦ip 漂移，会导致流量502.但在云原生环境中，发生ip漂移是非常常见的。

* 2. server 变更会导致reload
添加host


# 1. nginx 如何实现不reload 更新upstream

一个典型


# 2. nginx 如何实现不reload 更新server


# 3. apisix 如何实现不reload 更新upstream & server
