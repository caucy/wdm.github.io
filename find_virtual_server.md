## nginx 到底用哪个server 块？

### 背景
我们使用的nginx是定制化的nginx，不同于原生nginx，对ssl和h2支持做了很大的改动。我们会遇到几个问题，

* 1. 带sni，我们会命中哪个server 
* 2. 不带sni, 我们会命中哪个server？
* 3. 我们是不是能让一个端口的不同server 是否开启h2 互不干扰
* 4. 如果我们想实现不同server 块互不干扰开启h2 该如何实现

### 如何发现命中了不对的server

比较明显的是，如果没开h2,会404, virtual server 开了h2, 但是default server 没开，会命中h2 连h1 的错。
有类似错:

```
 rpc error: code = Unimplemented desc = unexpected HTTP status code received from server: 404 (Not Found); transport: received unexpected content-type "text/html"
```

### nginx 如何觉得用哪个server 块的？

nginx 通过 [ngx_http_set_virtual_server](https://github.com/nginx/nginx/blob/bfc5b35827903a3c543b58e4562db8b62021c164/src/http/ngx_http_request.c#L2191) 觉得选哪个server 块。


```
ngx_http_set_virtual_server(ngx_http_request_t *r, ngx_str_t *host)
{
    ngx_int_t                  rc;
    ngx_http_connection_t     *hc;
    ngx_http_core_loc_conf_t  *clcf;
    ngx_http_core_srv_conf_t  *cscf;

#if (NGX_SUPPRESS_WARN)
    cscf = NULL;
#endif

    hc = r->http_connection;

#if (NGX_HTTP_SSL && defined SSL_CTRL_SET_TLSEXT_HOSTNAME)

    if (hc->ssl_servername) {
        if (hc->ssl_servername->len == host->len
            && ngx_strncmp(hc->ssl_servername->data,
                           host->data, host->len) == 0)
        {
#if (NGX_PCRE)
            if (hc->ssl_servername_regex
                && ngx_http_regex_exec(r, hc->ssl_servername_regex,
                                          hc->ssl_servername) != NGX_OK)
            {
                ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                return NGX_ERROR;
            }
#endif
            return NGX_OK;
        }
    }

#endif

    rc = ngx_http_find_virtual_server(r->connection,
                                      hc->addr_conf->virtual_names,
                                      host, r, &cscf);

    if (rc == NGX_ERROR) {
        ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return NGX_ERROR;
    }

#if (NGX_HTTP_SSL && defined SSL_CTRL_SET_TLSEXT_HOSTNAME)

    if (hc->ssl_servername) {
        ngx_http_ssl_srv_conf_t  *sscf;

        if (rc == NGX_DECLINED) {
            cscf = hc->addr_conf->default_server;
            rc = NGX_OK;
        }

        sscf = ngx_http_get_module_srv_conf(cscf->ctx, ngx_http_ssl_module);

        if (sscf->verify) {
            ngx_log_error(NGX_LOG_INFO, r->connection->log, 0,
                          "client attempted to request the server name "
                          "different from the one that was negotiated");
            ngx_http_finalize_request(r, NGX_HTTP_MISDIRECTED_REQUEST);
            return NGX_ERROR;
        }
    }

#endif

    if (rc == NGX_DECLINED) {
        return NGX_OK;
    }

    r->srv_conf = cscf->ctx->srv_conf;
    r->loc_conf = cscf->ctx->loc_conf;

    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

    ngx_set_connection_log(r->connection, clcf->error_log);

    return NGX_OK;
}
```
上面这段代码的意思，主要是设置request.clcf 的srv_conf的时候。如果有sni, 会用sni 找virtual_server，找到了就用 virtual_server，找不到命中default_server。如果没有sni,
会用host 去找virtual_server，如果找到了直接使用，找不到用default server.

### 如何实现让一个端口的不同server 是否开启h2 互不干扰？
未完待续


