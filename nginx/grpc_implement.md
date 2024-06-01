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

## 3. http v1 request 的处理流程
```
-- ngx_http_init_connection
  -- ngx_http_wait_request_handler
    -- ngx_http_process_request_line
      -- ngx_http_parse_request_line 直到header 处理完
        -- ngx_http_process_request_headers
          -- ngx_http_process_request
            -- 11 个header 阶段
              -- ngx_http_proxy_handler 开始发送数据给upstream
                -- ngx_http_upstream_init
                  -- r->read_event_handler = ngx_http_upstream_read_request_handler

```
**ngx_http_upstream_read_request_handler**  是重入点，不断收request 数据

## 3. http v2协议request 的hook 流程

**重入的入口是ngx_http_v2_read_handler**/

数据的读都在这个函数，判断frame 是否完成在状态机完成

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

state.handler 初始化是 ngx_http_v2_state_preface，然后调用ngx_http_v2_state_headers，
获取frame 头会根据请求frame 类型调用不同的状态回调
```
static ngx_http_v2_handler_pt ngx_http_v2_frame_states[] = {
    ngx_http_v2_state_data,               /* NGX_HTTP_V2_DATA_FRAME */
    ngx_http_v2_state_headers,            /* NGX_HTTP_V2_HEADERS_FRAME */
    ngx_http_v2_state_priority,           /* NGX_HTTP_V2_PRIORITY_FRAME */
    ngx_http_v2_state_rst_stream,         /* NGX_HTTP_V2_RST_STREAM_FRAME */
    ngx_http_v2_state_settings,           /* NGX_HTTP_V2_SETTINGS_FRAME */
    ngx_http_v2_state_push_promise,       /* NGX_HTTP_V2_PUSH_PROMISE_FRAME */
    ngx_http_v2_state_ping,               /* NGX_HTTP_V2_PING_FRAME */
    ngx_http_v2_state_goaway,             /* NGX_HTTP_V2_GOAWAY_FRAME */
    ngx_http_v2_state_window_update,      /* NGX_HTTP_V2_WINDOW_UPDATE_FRAME */
    ngx_http_v2_state_continuation        /* NGX_HTTP_V2_CONTINUATION_FRAME */
};
```
header frame 处理的流程
```
-- ngx_http_v2_read_handler
  -- ngx_http_v2_state_preface
    -- ngx_http_v2_state_head
        -- h2c->state.hanlder = ngx_http_v2_state_headers
        -- ngx_http_v2_state_header_complete
        -- ngx_http_v2_run_request()-> ngx_http_process_request -> 11个阶段
        --ngx_http_v2_state_complete
          -- h2c->state->hanlder = ngx_http_v2_state_head 回到探测frame header 的回调去

```

data frame 处理流程
```
-- ngx_http_v2_read_handler
  -- ngx_http_v2_state_preface
    -- ngx_http_v2_state_head
        -- h2c->state.hanlder = ngx_http_v2_state_data
          -- ngx_http_v2_state_read_data // 真实read data, 长度没收完会设置handler = ngx_http_v2_state_read_data
            --ngx_http_v2_state_complete
              -- handler = ngx_http_v2_state_read_data // 数据没收完
              -- h2c->state->hanlder = ngx_http_v2_state_head // 数据收完了回到探测frame header 的回调去


-- 如果request 的header 11阶段处理完了，会调用proxy_module_handler or grpc_module_handler

  -- ngx_http_v2_state_read_data
    -- if(r->request_body){ngx_http_v2_process_request_body()}
      -- post_handler 执行read_request_client_body 对应的回调，开始发数据到upstream
    -- h2c->state->hanlder = ngx_http_v2_state_head

```


## 3. http2 协议response 的hook 流程


