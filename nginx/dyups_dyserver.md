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

主要的实现：
```
static ngx_int_t
ngx_dyups_do_update(ngx_str_t *name, ngx_buf_t *buf, ngx_str_t *rv)
{
    ngx_int_t                       rc, idx;
    ngx_http_dyups_srv_conf_t      *duscf;
    ngx_http_dyups_main_conf_t     *dumcf;
    ngx_http_upstream_srv_conf_t  **uscfp;
    ngx_http_upstream_main_conf_t  *umcf;

    umcf = ngx_http_cycle_get_module_main_conf(ngx_cycle,
                                               ngx_http_upstream_module); // 获取全局upstrem conf
    dumcf = ngx_http_cycle_get_module_main_conf(ngx_cycle,
                                                ngx_http_dyups_module); //获取dyups main conf

    duscf = ngx_dyups_find_upstream(name, &idx);// 生成dyups server conf  
    if (duscf) {
        ngx_log_error(NGX_LOG_DEBUG, ngx_cycle->log, 0,
                      "[dyups] upstream reuse, idx: [%i]", idx);

        if (!duscf->deleted) {
            ngx_log_error(NGX_LOG_DEBUG, ngx_cycle->log, 0,
                          "[dyups] upstream delete first");
            ngx_dyups_mark_upstream_delete(duscf);

            duscf = ngx_dyups_find_upstream(name, &idx);

            ngx_log_error(NGX_LOG_DEBUG, ngx_cycle->log, 0,
                          "[dyups] find another, idx: [%i]", idx);
        }
    }

    if (idx == -1) {
        /* need create a new upstream */

        ngx_log_error(NGX_LOG_INFO, ngx_cycle->log, 0,
                      "[dyups] create upstream %V", name);

        duscf = ngx_array_push(&dumcf->dy_upstreams);
        if (duscf == NULL) {
            ngx_str_set(rv, "out of memory");
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }

        uscfp = ngx_array_push(&umcf->upstreams);
        if (uscfp == NULL) {
            ngx_str_set(rv, "out of memory");
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }

        ngx_memzero(duscf, sizeof(ngx_http_dyups_srv_conf_t));
        idx = umcf->upstreams.nelts - 1;
    }

    duscf->idx = idx;
    rc = ngx_dyups_init_upstream(duscf, name, idx); // dyups server conf 初始化

    if (rc != NGX_OK) {
        ngx_str_set(rv, "init upstream failed");
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    /* init upstream */
    rc = ngx_dyups_add_server(duscf, buf);
    if (rc != NGX_OK) {
        ngx_str_set(rv, "add server failed");
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    ngx_str_set(rv, "success");

    return NGX_HTTP_OK;
}
```

# 2. nginx 如何实现不reload 更新server


# 3. apisix 如何实现不reload 更新upstream & server
