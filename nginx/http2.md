# nginx 如何实现http2 转发的

## 先抛几个问题
1. nginx 处理http2请求，回源的时候是多路复用的吗？
2. 如何判断一个http2/grpc message 接受完了？
3. 一个端口同时有h1 h2, 什么时候走h2?
4. 如何判断流结束？
5. http2 的处理流程跟http1 有什么联系跟区别？会走11个阶段吗？
6. http3 的处理流程跟http2 的实现有联系吗？


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
5. h2->stream 是一个数组，可以根据streams_index 找到stream, 一个stream 由一组相同stream_id 的frame 组成

Frame Format：
All frames begin with a fixed 9-octet header followed by a variable-length payload.
```
 +-----------------------------------------------+
 |                 Length (24)                   |
 +---------------+---------------+---------------+
 |   Type (8)    |   Flags (8)   |
 +-+-------------+---------------+-------------------------------+
 |R|                 Stream Identifier (31)                      |
 +=+=============================================================+
 |                   Frame Payload (0...)                      ...
 +---------------------------------------------------------------+
```

**如何判断流结束？**
stream flags 里面有两个重要标志位，一个是压缩，一个end

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

**request_body_no_buffering** on 模式下，先调用**ngx_http_upstream_init**，后续r->connection 来数据后再调用

**ngx_http_upstream_read_request_handler**，然后发给upstream->connection.

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

**header frame 处理的流程：**

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
**h2 把每个stream 拆成了一个request, 多路复用的request在链接upstream的时候会变成多个并行的请求!!!**

这里有个难理解的点，11个阶段最后一个handler 是proxy_module_handler, 会调用 **ngx_http_read_client_request_body**, unbuffer request 会立刻调用ngx_http_upstream_init。

```
-- ngx_http_read_client_request_body
  -- 初始化r->request_body
  -- if r->stream => ngx_http_v2_read_request_body 
    -- ngx_http_v2_send_window_update 调整window size
    -- post_handler()执行upstream_init
```

**ngx_http_read_client_request_body** 处理完后，h2c 来数据后，ngx_http_v2_read_handler会被触发，然后调用到r 的read_event_handler去。

**data frame 处理流程:**
```
-- ngx_http_v2_read_handler
  -- ngx_http_v2_state_preface
    -- ngx_http_v2_state_head
        -- h2c->state.hanlder = ngx_http_v2_state_data
          -- ngx_http_v2_state_read_data
              -- ngx_http_v2_process_request_body
                --...
                -- post_event(fake_connection_read), 此时stream 的所有数据在r->request_body->buf 里面。fake_connection->read_event_hanlder = ngx_http_request_handler
            --ngx_http_v2_state_complete
              -- handler = ngx_http_v2_state_read_data // 数据没收完
                -- h2c->state->hanlder = ngx_http_v2_state_head // 数据收完了回到探测frame header 的回调去
```
h2c 收到data frame数据后，此时post_event fake_connection->read_event_hanlder = **ngx_http_request_handler**，最终到**ngx_http_upstream_read_request_handler**去，后面流程和http1一致。


## 5. http2 协议response处理流程

响应的流程比请求的流程要简单，核心在**ngx_http_v2_header_filter 中替换了send_chain = ngx_http_v2_send_chain**

```
  -- ngx_http_upstream_handler
    -- **header_filter阶段**
      -- u->read_event_handler = ngx_http_upstream_process_header
        --......
        --ngx_http_v2_header_filter
          -- send_chain = ngx_http_v2_send_chain

    -- **body_filter阶段**
    --ngx_http_upstream_send_response
      -- u->read_event_handler = ngx_http_upstream_process_non_buffered_upstream
        -- 各种body_filter
        -- ngx_write_filter
          -- send_chain() -- ngx_http_v2_send_chain // 将frame 挂到h2c->last_out去并发出去
  
```
**ngx_http_upstream_process_header** 和**ngx_http_upstream_process_non_buffered_upstream** 是header 和response
重入的入口。

ngx_http_v2_send_chain 将frame 挂到h2c->last_out去并发出去，ngx_http_v2_state_read_data也会检查h2c->last_out并发送

## 6. grpc 和普通http2 有什么不同？
* grpc 处理request 跟h2没有区别，处理upstream有几个不同：
1. grpc module 直接替换了clcf，所以content_phase 阶段不走 ngx_http_proxy_handler，调用grpc_handler
2. 访问upstream的时候，重新实现upstream的回调，封装成grpc格式
3. grpc 和quic 都默认设置了**request_body_no_buffering=1**，影响ngx_http_read_client_request_body，走非buffer模式

* grpc message 如何拆成http2 data frame的？
![image](https://github.com/caucy/wdm.github.io/assets/19687952/27541d0e-e0f5-49f4-9d20-3114d33cff42)
这里有一个细节，一个grpc message, 如果超出一定大小，会被拆成多个http2 frame,  **message length 只在第一个data frame 中有**。

## 7.grpc 处理upstream读的处理流程
```
-- ngx_http_upstream_handler
    -- u->header_process(处理header frame, 放到r中)
        -- input_filter = ngx_http_grpc_filter
        -- header_filter
            --grpc_header_filter
        -- body_filter
            -- write_filter
            -- out_filter = grpc_out_filter
```
1. **ngx_http_grpc_filter** 处理goaway 或者setting frame 等，将u->buffer 数据**攒完一个data frame**后放到u->out_bufs去

2. **ngx_http_grpc_parse_frame** 是非常重要的一个函数，按h2 frame 解析frame, 没解析完就返回again。

##  回顾下nginx 要实现http2，要实现哪些重要的hook点
1. h2c数据来了要有读回调，回调是**ngx_http_v2_read_handler**, 实现关联stream, 然后关联http request 的逻辑
2. request 级别来数据了，需要识别header 是否收完，调**ngx_http_v2_run_request**走11个阶段
3. http2 是unbuffer的，所以read_client_request_body 需要兼容，调用upstream_init后，**ngx_http_v2_read_handler**要实现发送数据给upstream的逻辑，最终到**ngx_http_upstream_read_request_handler**去
4. http1 数据需要转成http2, 所以需要header_filter 阶段换掉r->send_chain，用h2的格式在write_body_filter后发送数据给客户端

nginx 实现quic 的几个重要的函数基本跟h2一样
