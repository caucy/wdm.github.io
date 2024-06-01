# nginx 如何实现http2 转发的

## 1. http2 对连接的处理
没有配ssl
```
--accept
  -- ngx_http_init_connection
    -- rev_hanlder = ngx_http_v2_init

```

配置了ssl
```
-- ngx_http_ssl_handshake_handler
  -- h2 on & SSL_get0_alpn_selected == h2 (开了h2 ,同时alpn 中有h2)
  -- ngx_http_v2_init
    -- rev->handler = ngx_http_v2_read_handler
    -- rev->handler = ngx_http_v2_write_handler
```

ngx_http_v2_read_handler 实现循环接受client 的请求，里面有状态机，判断请求frame 类型，bing执行对应回调
ngx_http_v2_write_handler 将h2c->last_out 链表的数据发给client

## 2. 重要的数据结构
几个重要的数据结构
1. h2mcf h2的核心conf，一系列参数
2. h2c 代表一个h2 对象，有多个stream，每个stream 对应一个http_request 对象，一个fake connection
3. h2c->state.buffer 缓存链接的数据
4. h2->connection 跟client 真实的连接, r->stream->h2c->connection 可以找到真正的connection
5. h2->stream 是一个数组，可以根据streams_index 找到stream


## 3. http协议request 的hook 流程
/* 重入的入口是ngx_http_v2_read_handler */

```

    do {
        p = h2mcf->recv_buffer;

        ngx_memcpy(p, h2c->state.buffer, NGX_HTTP_V2_STATE_BUFFER_SIZE);
        end = p + h2c->state.buffer_used;

        n = c->recv(c, end, available);
        。。。

        do {
            p = h2c->state.handler(h2c, p, end);

            if (p == NULL) {
                return;
            }

        } while (p != end);

        h2c->total_bytes += n;

        if (h2c->total_bytes / 8 > h2c->payload_bytes + 1048576) {
            ngx_log_error(NGX_LOG_INFO, c->log, 0, "http2 flood detected");
            ngx_http_v2_finalize_connection(h2c, NGX_HTTP_V2_NO_ERROR);
            return;
        }

    } while (rev->ready);
```



## 3. grpc 协议response 的hook 流程


