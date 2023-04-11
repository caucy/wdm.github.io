## nginx 到底用哪个server 块？

### 背景
我们使用的nginx是定制化的nginx，不同于原生nginx，对ssl和h2支持做了很大的改动。我们会遇到几个问题，

* hash_t, hash_init_t, wildcard_t, combind_hash_t 区别是什么?
* 带sni，我们会命中哪个server 
* 不带sni, 我们会命中哪个server？

### server 查找几个hash 结构的区别

```
typedef struct {
    ngx_hash_t            hash; // 正常hash
    ngx_hash_wildcard_t  *wc_head; // 前缀hash
    ngx_hash_wildcard_t  *wc_tail; // 后缀hash
} ngx_hash_combined_t;
```
一般混合hash 包含了普通hash 和前缀、后缀hash，ngx_hash_combined_t 等三种hash 都借hash_init_t 初始化。

主要api
```
// 初始化
ngx_hash_init(&hash_init, hash_array.keys.elts, hash_array.keys.nelts) hash_array 可以是hash_t 数组，或者ngx_hash_wildcard_t 数组

// 查找
ngx_hash_find_combined(&combinedHash, h, (u_char *)k6.data, k6.len)
ngx_hash_find(&hash, h, (u_char *)k3.data, k3.len);

```


demo:
```
#include <ngx_core.h>

int main (int argc, char *argv[])
{
    ngx_int_t rc;
    ngx_pool_t *pool1, *pool2;
    ngx_hash_init_t hash_init;
    ngx_hash_keys_arrays_t hash_array;
    
    ngx_hash_combined_t combinedHash;
    
    //ngx_memzero(&hash_array, sizeof(ngx_hash_keys_arrays_t));
    //ngx_memzero(&combinedHash, sizeof(ngx_hash_combined_t));
	
	pool1 = ngx_create_pool(NGX_DEFAULT_POOL_SIZE, NULL);
    if (pool1 == NULL) {
        perror("ngx_create_pool() failed.");
        return 1;
    }
    pool2 = ngx_create_pool(NGX_DEFAULT_POOL_SIZE, NULL);
    if (pool2 == NULL) {
        perror("ngx_create_pool() failed.");
        return 1;
    }
	//bug
	//char *k2 = "test";
    
    ngx_str_t k3;
    k3.len = ngx_strlen("www.baidu.com");
    k3.data = ngx_pcalloc(pool1, k3.len);
    ngx_memcpy(k3.data, "www.baidu.com", k3.len);
    
    //注意，通配符key必须可以修改，所以不能使用静态的数据
    ngx_str_t k4;
    k4.len = ngx_strlen("*.baidu.com");
    k4.data = ngx_pcalloc(pool1, k4.len);
    ngx_memcpy(k4.data, "*.baidu.com", k4.len);
    
    ngx_str_t k5;
    k5.len = ngx_strlen("www.freecls.*");
    k5.data = ngx_pcalloc(pool1, k5.len);
    ngx_memcpy(k5.data, "www.freecls.*", k5.len);
    
    char *v3 = "www.baidu.com";
    char *v4 = "*.baidu.com";
	char *v5 = "www.freecls.*";
	
	//char *k2 = "1111";

    ngx_cacheline_size = NGX_CPU_CACHE_LINE;

    
    hash_array.pool = pool1;
    hash_array.temp_pool = pool2;
    
    if (ngx_hash_keys_array_init(&hash_array, NGX_HASH_SMALL) != NGX_OK) {
        return NGX_ERROR;
    }
    
    rc = ngx_hash_add_key(&hash_array, &k3, v3, NGX_HASH_READONLY_KEY);
    if (rc == NGX_ERROR) {
        perror("ngx_hash_add_key() failed.");
        return 1;
    }
    
    rc = ngx_hash_add_key(&hash_array, &k4, v4, NGX_HASH_WILDCARD_KEY);
    if (rc == NGX_ERROR) {
        perror("ngx_hash_add_key() failed.");
        return 1;
    }
    
    rc = ngx_hash_add_key(&hash_array, &k5, v5, NGX_HASH_WILDCARD_KEY);
    if (rc == NGX_ERROR) {
        perror("ngx_hash_add_key() failed.");
        return 1;
    }

    
    hash_init.key         = ngx_hash_key_lc;
    hash_init.max_size    = 128;
    hash_init.bucket_size = 64;
    hash_init.name        = "hash_array_name";
    hash_init.pool        = pool1;

	if(hash_array.keys.nelts){
		hash_init.hash        = &combinedHash.hash;
		hash_init.temp_pool   = NULL;
		if (ngx_hash_init(&hash_init, hash_array.keys.elts, hash_array.keys.nelts) != NGX_OK) {
        	return 1;
    	}
	}
    
    if(hash_array.dns_wc_head.nelts){
		hash_init.hash        = NULL;
		hash_init.temp_pool   = hash_array.temp_pool;
		if (ngx_hash_wildcard_init(&hash_init, hash_array.dns_wc_head.elts, hash_array.dns_wc_head.nelts) != NGX_OK) {
        	return 1;
    	}
    	
    	combinedHash.wc_head = (ngx_hash_wildcard_t *)hash_init.hash;
	}
	
	if(hash_array.dns_wc_tail.nelts){
		hash_init.hash        = NULL;
		hash_init.temp_pool   = hash_array.temp_pool;
		if (ngx_hash_wildcard_init(&hash_init, hash_array.dns_wc_tail.elts, hash_array.dns_wc_tail.nelts) != NGX_OK) {
        	return 1;
    	}
    	
    	combinedHash.wc_tail = (ngx_hash_wildcard_t *)hash_init.hash;
	}
    
    

	ngx_uint_t h;
	
	//查找利用通配符查找
	ngx_str_t k6 = ngx_string("a.baidu.com");
	h = ngx_hash_key_lc((u_char *)k6.data, k6.len);
    u_char *r1 = ngx_hash_find_combined(&combinedHash, h, (u_char *)k6.data, k6.len);
    if (r1 == NULL) {
        r1 = "not find";
    }else{
    	printf("k6: %s\n", r1);
    }
    
    ngx_str_t k7 = ngx_string("www.baidu.com");
	h = ngx_hash_key_lc((u_char *)k7.data, k7.len);
    r1 = ngx_hash_find_combined(&combinedHash, h, (u_char *)k7.data, k7.len);
    if (r1 == NULL) {
        r1 = "not find";
    }else{
    	printf("k7: %s\n", r1);
    }
    
    ngx_str_t k8 = ngx_string("www.freecls.com");
	h = ngx_hash_key_lc((u_char *)k8.data, k8.len);
    r1 = ngx_hash_find_combined(&combinedHash, h, (u_char *)k8.data, k8.len);
    if (r1 == NULL) {
        r1 = "not find";
    }else{
    	printf("k8: %s\n", r1);
    }
    
    ngx_str_t k9 = ngx_string("www.freecls.cn");
	h = ngx_hash_key_lc((u_char *)k9.data, k9.len);
    r1 = ngx_hash_find_combined(&combinedHash, h, (u_char *)k9.data, k9.len);
    if (r1 == NULL) {
        r1 = "not find";
    }else{
    	printf("k9: %s\n", r1);
    }
	
	
    return 0;
}
```


### 两个hash 的demo


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


