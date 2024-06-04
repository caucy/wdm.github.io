# quic 在nignx 的实现
 
## 1. 重要的数据结构
 ngx_xquic_recv_packet_t 
    里面有一次read的原始接受的udp 数据，buf 最大1500

xqc_engine_t conns_hash 维护所有的连接xqc_connection_t
 
xqc_connection_t 一个抽象的quic 连接接通过cid 串联， user_data 是ngx_http_xquic_connection_t有真正的udp connection
 
xqc_stream_s stream_data_in 存frame 数据

xqc_h3_stream_t 包含一个xqc_stream_s，
 
 cid和dcid 的区别？CID 主要由客户端维护，DCID 主要由服务器维护。
 quic 必须走tls？看代码是的
 xqc_conn_server_init_path_addr 初始化path list 是什么？path 参数默认是0

## 2. 连接初始流程
 
-- ngx_xquic_process_init 进程启动阶段
    -- rev->handler = ngx_xquic_event_recv; bind的fd 添加read 回调
        
bind fd有read event 被触发，创建quic connection
 -- ngx_xquic_event_recv
     -- ngx_xquic_dispatcher_process_packet 将buf 关联cid 的数据结构（每个udp 包都有cid 位）
         -- xqc_engine_packet_process 处理每个package，设置
             -- xqc_conn_server_create 
                 -- **ngx_xquic_conn_accept** 作用：创建xqc_connection_t 
                     -- ngx_http_v3_create_connection
                         -- ngx_http_xquic_init_connection
                             -- c->read->handler = ngx_http_xquic_readmsg_handler;(后面给换成ngx_http_xquic_read_handler)
                             -- c->write->handler = ngx_http_xquic_write_handler;
                     -- xqc_conn_set_transport_user_data（绑定xqc_connection_t 和ngx_http_xquic_connection_t）
                     -- xqc_conn_set_alp_user_data

## 3. header/body读写处理
    -- ngx_http_xquic_read_handler (读fd 的数据)
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
            -- **xqc_h3_stream_process_request** (识别出frame 是data 还是header 还是push)
                -- xqc_h3_request_on_recv_header
                    -- **ngx_http_v3_request_read_notify**(处理header body)
                        -- ngx_http_v3_state_headers_complete
                            -- ngx_http_v3_run_request
                                -- ngx_http_handler 开始走11个nginx 阶段
