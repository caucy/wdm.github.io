# xquic 在nignx 的实现
 
## 1. 重要的数据结构

xqc_engine_t conns_hash 维护所有的连接xqc_connection_t
 
xqc_connection_t 一个抽象的quic连接, 通过cid 定义
 
xqc_stream_s 一个stream 关联一个request, 存在一个fake connection， 一个stream 含有多个frame

xqc_h3_stream_t 包含一个xqc_stream_s


## 2. 连接初始化流程
```
  -- ngx_xquic_process_init 进程启动阶段
  
    -- rev->handler = ngx_xquic_event_recv; bind的fd 添加read 回调
    
bind fd有read event 被触发，创建quic connection
   -- ngx_xquic_event_recv
   
     -- ngx_xquic_dispatcher_process_packet 将buf 关联cid 的数据结构（每个udp 包都有cid 位）
     
         -- xqc_engine_packet_process 处理每个package
         
             -- xqc_conn_server_create 
             
                 ngx_xquic_conn_accept 创建xqc_connection_t
                 
                     -- ngx_http_v3_create_connection
                     
                         -- ngx_http_xquic_init_connection
                         
                             -- c->read->handler = ngx_http_xquic_readmsg_handler;(后面给换成ngx_http_xquic_read_handler)
                             -- c->write->handler = ngx_http_xquic_write_handler;
                             
                     -- xqc_conn_set_transport_user_data（绑定xqc_connection_t 和ngx_http_xquic_connection_t）
                     
                     -- xqc_conn_set_alp_user_data
```

**ngx_http_xquic_init_connection** 会给udp socket 挂一个读写回调

**ngx_xquic_recv_packet_t** 里面有一次read的原始接受的udp 数据，buf 最大1500

## 3. header/body读写处理
```
    -- ngx_http_xquic_read_handler 读fd 的数据
        -- ngx_http_xquic_session_process_packet（复用cid）
            -- xqc_engine_packet_process
                -- xqc_engine_main_logic_internal
                    -- xqc_engine_main_logic
                        -- xqc_engine_process_conn
                            -- xqc_process_read_streams
                             -- xqc_h3_stream_read_notify
                                -- xqc_h3_stream_process_data
                                    -- xqc_h3_stream_process_in
    -- xqc_h3_stream_process_bidi
        -- xqc_h3_stream_process_bidi_payload
            -- xqc_h3_stream_process_request (识别出frame 是data 还是header 还是push)
                -- xqc_h3_request_on_recv_header
                    -- **ngx_http_v3_request_read_notify**(处理header body)
                        -- ngx_http_v3_state_headers_complete
                            -- ngx_http_v3_run_request
                                -- ngx_http_handler 开始走11个nginx 阶段
```
**xqc_h3_stream_process_request** 会识别出stream 的frame 是data，header 还是push

应用程序自己实现收udp **ngx_http_xquic_read_handler**，自己读fd；

**ngx_http_v3_request_read_notify** 需要自己从stream中解应用层数据，区分应用层的header 还是body 是否收发完毕。

一般集群都是默认buffer request，**ngx_http_v3_process_request_body** 会回调回upstream 的read_client_request_body 的post_handler 也就是upstream_init，然后转发数据到upstream

## 4. upstream 来的数据如何通过quic 发出去？
upstream来的数据比较简单， header_filter 阶段替换掉send_chain, 实现一个send_quic_chain即可。

## 5. 回顾下nginx 要实现http3，要实现哪些重要的hook点
1. 进程启动的时候要实现bind 逻辑，逻辑在**ngx_xquic_event_recv** 中
2. udp 数据来了要有读回调，逻辑在**ngx_http_xquic_read_handler**, 这个函数要实现类似h2的，关联stream, 然后关联http request 的逻辑
3. request 级别来数据了，需要识别header 是否收完，body 是否收完，会有回调**ngx_http_v3_request_read_notify**
4. request 收完body 要调用core_run_phase, 走http1的11个阶段，逻辑在**ngx_http_v3_run_request**
5. http3 是unbuffer的，所以**read_client_request_body** 需要兼容，直接调用upstream_init，然后设置r->read_event_handler
6. http1 数据需要转成http3, 所以需要header_filter 阶段换掉r->send_chain，用quic 的格式在write_body_filter后发送数据给客户端

