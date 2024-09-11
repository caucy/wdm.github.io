## 文章列表

### nginx 开发
[nginx 编译成静态库调试自己的代码](nginx/nginx_as_static.md)

[nginx 使用哪个server](nginx/find_virtual_server.md)

[lua 性能分析](nginx/lua_performance.md)

[apisix & nginx 如何实现变更配置不reload](nginx/dyups_dyserver.md)

[nginx 共享内存的常见使用方式和原理](nginx/shm_example.md)

[nginx lua 交互一些常见写法](nginx/lua_c.md)

[参考nginx-lua 实现replace reqeust body](nginx/repalce_request_body.md)

[参考nginx-lua 实现body-filter replace response body](nginx/replace_response_body.md)

[4 7 层如何做ip传递的](nginx/server_addr.md)

[shutdown 为啥一直不退出？](nginx/shutdown.md)

[nginx 如何判断tls的时候走h1, h2 ?](nginx/ssl_h2.md)

[nginx 实现tls session ticket key 共享](nginx/session_ticket.md)

[nginx 实现keyless 减少ssl握手开销](nginx/keyless.md)

### envoy
[envoy 如何解析http 协议](envoy/http_parser.md)

[envoy 如何编译](envoy/build.md)

[envoy 如何使用常见的filter](envoy/use_filter.md)

[envoy 请求的流转过程](envoy/request.md)

[envoy 如何实现一个filter](envoy/implement_filter.md)


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


### rust 编程


### 问题定位思路

[如何定位c/c++的内存泄露](nginx/mem_leak.md)

[如何定位线上cpu热点](c/cpu_profile.md)

[定位use after free](c/asan.md)

[如果仅用tcpdump 找到指定的http包](c/tcpdump.md)

### ebpf
[如何使用ebpf]

[cilium 实现]

### 安全
[云高防如何防cc攻击](waf/cc.md)

[hyperscan 的使用](waf/hyperscan.md)

[一些常见的攻击的waf拦截手段](waf/waf.md)

[根据个人信息生成密码字典的方法](waf/passwd.md)

[sse 协议与流式接口安全](waf/sse.md)

[xss 攻击和绕过思路](waf/xss.md)

### 踩坑系列
[lua 奇葩的空洞数组](bug/lua_array.md)

[golang http transport 错误使用的泄漏](bug/golang_transport.md)

[resty dsl 的一些缺陷](bug/dsl.md)

[proxy_add_header/grpc_add_header被覆盖](nginx/add_header.md)

[三次握手阶段sync重传问题排查](nginx/syn.md)
