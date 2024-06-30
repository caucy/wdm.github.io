## 文章列表

### nginx 开发常用
[nginx 编译成静态库调试自己的代码](nginx/nginx_as_static.md)

[nginx 使用哪个server](nginx/find_virtual_server.md)

[lua 性能分析](nginx/lua_performance.md)

[apisix & nginx 如何实现变更配置不reload](nginx/dyups_dyserver.md)

[nginx 共享内存的常见使用方式和原理](nginx/shm_example.md)

[nginx lua 交互一些常见写法](nginx/lua_c.md)

[参考nginx-lua 实现replace reqeust body](nginx/repalce_request_body.md)

[参考nginx-lua 实现body-filter replace response body](replace_response_body.md)

[4 7 层如果做ip传递的](nginx/server_addr.md)

[shutdown 为啥一直不退出？](nginx/shutdown.md)

[nginx 如何判断tls的时候走h1, h2 ?](nginx/ssl_h2.md)

[nginx 实现tls session ticket key 共享](nginx/session_ticket.md)

[nginx 实现keyless 减少ssl握手开销](nginx/keyless.md)


### k8s
[常见debug 指令](k8s/debug.md)

[卷挂载冲突问题](k8s/volume_confict.md)

[如何实现autoport]



### ws 协议
[websocket的握手和解包](websocket/websocket_frame.md)

[websocket当4层代理跑私有协议](websocket/websocket_proxy.md)

[nginx chunked数据如何做安全防护](nginx/chunked.md)

[nginx ws 模块如何做安全防护](nginx/ws_waf.md)

### http2 + quic 协议

[nginx 如何实现http2 转发的](nginx/http2.md)

[对quic协议的一些理解](nginx/quic.md)

[nginx xquic协议集成的设计与实现](nginx/xquic.md)

[nginx chrome quic 设计与实现](nginx/chrome_quic.md)

### envoy
[envoy 如何解析http 协议](envoy/http_parser.md)

[envoy 如何编译](envoy/build.md)

[envoy 如何使用常见的filter](envoy/use_filter.md)

[envoy 如何实现一个filter](envoy/implement_filter.md)

### rust 编程


### 性能分析

[如何定位c/c++的内存泄露](nginx/mem_leak.md)

[如何定位线上cpu热点](c/cpu_profile.md)

### ebpf
[如何使用ebpf]

[cilium 实现]

### 安全
[云高防如何防cc攻击](waf/cc.md)

[云waf的实现](waf/waf.md)

### 踩坑系列
[lua 奇葩的空洞数组](bug/lua_array.md)

[golang http transport 错误使用的泄漏](bug/golang_transport.md)

[resty dsl 的in是 on 遍历](bug/dsl.md)
