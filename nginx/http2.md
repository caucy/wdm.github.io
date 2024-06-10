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

ngx_http_v2_read_handler 实现循环接受client 的请求，里面有状态机，判断请求frame 类型，并执行对应回调
ngx_http_v2_write_handler 将h2c->last_out 链表的数据发给client

## 2. 重要的数据结构
几个重要的数据结构
1. h2mcf h2的核心conf，一系列参数
2. h2c 代表一个h2 对象，有多个stream，每个stream 对应一个http_request 对象，一个fake connection
3. h2c->state.buffer 缓存链接的数据
4. h2->connection 是client 真实的连接,因为多路复用关联了多个fake connection, r->stream->h2c->connection 可以找到真正的connection
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
                -- ngx_http_read_client_request_body 
                  -- ngx_http_upstream_init
                    ......
                    -- request_body_no_buffering=on =>  r->read_event_handler = ngx_http_upstream_read_request_handler

```
ngx_http_read_client_request_body默认读完body才调用ngx_http_upstream_init, 但是受**request_body_no_buffering**影响。

**request_body_no_buffering** on 模式下，先调用**ngx_http_upstream_init**，后续r->connection 来数据后再调用**ngx_http_upstream_read_request_handler**
读，然后发给upstream->connection.

如果request_body_no_buffering off, 会缓存request，收完body后调用**ngx_http_upstream_init**。


## 4. http v2协议request 的处理流程

核心函数是**ngx_http_v2_read_handler**,收数据后关联request， **ngx_http_v2_send_output_queue** 发送给client

数据的读都在**ngx_http_v2_read_handler**这个函数，判断frame 是否完成在状态机完成

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

state.handler 初始化是 **ngx_http_v2_state_preface**，然后调用**ngx_http_v2_state_headers**，
获取frame 头几个字节，根据请求frame 类型调用不同的状态回调

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

/**header frame 处理的流程：**/

```
-- ngx_http_v2_read_handler
  -- ngx_http_v2_state_preface
    -- ngx_http_v2_state_head
        -- h2c->state.hanlder = ngx_http_v2_state_headers
        -- ngx_http_v2_create_stream 创建流
          -- 创建fake connection + request
        -- ngx_http_v2_state_header_complete
          -- ngx_http_v2_run_request() -> ngx_http_process_request
            -- ...  11个阶段
            -- ngx_http_proxy_handler
              -- ngx_http_read_client_request_body
        --ngx_http_v2_state_complete
          -- h2c->state->hanlder = ngx_http_v2_state_head 回到探测frame header 的回调去
```

这里有个难理解的点，11个阶段最后一个handler 是proxy_module_handler, 会调用 **ngx_http_read_client_request_body**, unbuffer request 会立刻调用ngx_http_upstream_init。

```
-- ngx_http_read_client_request_body
  -- 初始化request_body
  -- if r->stream -> ngx_http_v2_read_request_body 
    -- ngx_http_v2_send_window_update 调整window size
    -- post_handler()执行upstream_init, r->read_event_handler = block_handler
```

**ngx_http_read_client_request_body** 处理完后，h2c 来数据后，ngx_http_v2_read_handler会被触发，然后调用到r 的read_event_handler去。

/**data frame 处理流程:**/
```
-- ngx_http_v2_read_handler
  -- ngx_http_v2_state_preface
    -- ngx_http_v2_state_head
        -- h2c->state.hanlder = ngx_http_v2_state_data
          -- ngx_http_v2_state_read_data
              -- ngx_http_v2_process_request_body
                --...
                -- post_event(fake_connection_read), 此时stream 的所有数据在r->request_body->buf 里面
            --ngx_http_v2_state_complete
              -- handler = ngx_http_v2_state_read_data // 数据没收完
                -- h2c->state->hanlder = ngx_http_v2_state_head // 数据收完了回到探测frame header 的回调去
```
h2c 收到数据后，通过post_event fake_connection->read_event_hanlder = **ngx_http_request_handler**，回调r->read_event_handler到**ngx_http_upstream_read_request_handler**去，后面流程和http1一致。

## 5. http2 协议response 的hook 流程

响应的流程比请求的流程要简单，ngx_http_v2_header_filter 中替换了send_chain = ngx_http_v2_send_chain

```
  -- ngx_http_upstream_handler
    -- ngx_http_upstream_process_header
      -- **header_filter阶段**
      --......
      --ngx_http_v2_header_filter
        -- send_chain = ngx_http_v2_send_chain

      -- **body_filter阶段**
      --ngx_http_upstream_send_response
        -- ngx_http_upstream_process_non_buffered_upstream
          -- 各种body_filter
          -- ngx_write_filter
            -- send_chain() -- ngx_http_v2_send_chain // 将frame 挂到h2c->last_out去并发出去
  
```

ngx_http_v2_send_chain 将frame 挂到h2c->last_out去并发出去，ngx_http_v2_state_read_data也会检查h2c->last_out并发送

## 6. http1 proxy_request_buffering 的处理流程回顾
```
-- ngx_http_proxy_handler 开始发送数据给upstream
  -- ngx_http_read_client_request_body
    -- ngx_http_upstream_init
      -- r->read_event_handler = ngx_http_upstream_read_request_handler
```
init_upstream 后 r->read_event_hanlder = **ngx_http_upstream_read_request_handler** 会先读request connection 的数据，再写给u->connection

```
-- ngx_http_upstream_read_request_handler
  -- ngx_http_upstream_send_request
    --ngx_http_upstream_send_request_body
      -- ngx_http_do_read_client_request_body
          先循环读r->connection, 然后ngx_output_chain 写数据给upstream

```
ngx_http_do_read_client_request_body 会识别是否buffer，同时识别h1, h2, h3


## 7. grpc 和普通http2 有什么不同？
grpc 处理request 跟h2没有区别，处理upstream有几个不同：
1. grpc module 直接替换了clcf，所以content_phase 阶段不走 ngx_http_proxy_handler，调用grpc_handler
2. 访问upstream的时候，重新实现upstream的回调，封装成grpc格式
3. grpc 和quic 都默认设置了**request_body_no_buffering=1**，跟proxy_request_buffer 没关系了

## 8.grpc 发起upstream处理流程
```
-- ngx_http_grpc_create_request 创建header frame
  -- 
```
