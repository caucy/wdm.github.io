## envoy 有哪些filter？
envoy 没有一个命令能dump 所有支持的filter，不是很友好，如果想看某个版本的envoy 支持哪些filter，可以看源码source/extensions/extensions_build_config.bzl，里面定义了所有的filter信息。

## envoy filter 如何使用
一个常见的filter 如下：
```
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 127.0.0.1, port_value: 10000 }
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          codec_type: AUTO
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route: { cluster: some_service }
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
```
几个问题需要回答
1. filter 的name是什么？
2. filter 的type 是什么含义？
3. filter 的参数如何查询？文档在哪，定义在哪里定义？
4. filter 的源码在哪，做了什么工作？

## envoy filter 使用指南
* filter 的name 一般定义在source/extensions/extensions_build_config.bzl
 
* type_config，一般默认type.googleapis.com 前缀，envoy.extensions.filters.http.router.v3 是pb package name, Router是type 类型
 
* filter 的官方文档
     https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/config
通过文档，可以查询每个filter 有哪些参数，每个参数的含义

* filter 的参数定义都在api/envoy/目录的pb文件中
以envoy.filters.network.http_connection_manager为例，参数定义在api/envoy/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto中
一般api/envoy/config 是通用的filter

## 常见的filter 配置
* 请求buffer：extensions.filters.http.buffer.v3.Buffer
```
http_filters:
  - name: envoy.filters.http.buffer
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.http.buffer.v3.Buffer
      max_request_bytes: 1024

```
请求大小超过1024byte 会返回413

* 压缩模块
```
- name: envoy.filters.http.compressor
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.http.compressor.v3.Compressor
      response_direction_config:
        common_config:
          min_content_length: 100
        disable_on_etag_header: true
      compressor_library:
        name: text_optimized
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.compression.gzip.compressor.v3.Gzip
          memory_level: 3
          window_bits: 10
```
response_direction_config 指定响应压缩配置，最小length 等，compressor_library 指定压缩算法等。

* header mutation header 修改
downstream header 修改
```
name: downstream-header-mutation
typed_config:
  "@type": type.googleapis.com/envoy.extensions.filters.http.header_mutation.v3.HeaderMutation
  mutations:
    request_mutations:
    - append:
        header:
          key: "downstream-request-global-flag-header"
          value: "downstream-request-global-flag-header-value"
        append_action: APPEND_IF_EXISTS_OR_ADD
    response_mutations:
    - append:
        header:
          key: "downstream-global-flag-header"
          value: "downstream-global-flag-header-value"
        append_action: APPEND_IF_EXISTS_OR_ADD
```

upstream header 修改
```
name: downstream-header-mutation-disabled-by-default
typed_config:
  "@type": type.googleapis.com/envoy.extensions.filters.http.header_mutation.v3.HeaderMutation
  mutations:
    request_mutations:
    - append:
        header:
          key: "downstream-request-global-flag-header-disabled-by-default"
          value: "downstream-request-global-flag-header-value-disabled-by-default"
        append_action: APPEND_IF_EXISTS_OR_ADD
    response_mutations:
    - append:
        header:
          key: "downstream-global-flag-header-disabled-by-default"
          value: "downstream-global-flag-header-value-disabled-by-default"
        append_action: APPEND_IF_EXISTS_OR_ADD
```
* lua 扩展extensions.filters.http.lua.v3.Lua
如果常见的filter 不能满足使用，可以在lua 扩展:
在流式传输的请求流和/或响应流中检查头部、正文和尾部。
修改头部和尾部。
阻塞并缓存整个请求/响应正文以进行检查。
对上游主机执行出站异步 HTTP 调用。可以在缓冲正文数据的同时执行此类调用，以便在调用完成时可以修改上游头部。
执行直接响应并跳过后续的过滤器迭代。例如，脚本可以向上游发起 HTTP 身份认证调用，然后直接响应 403 响应码。

envoy 提供了一系列[api](https://cloudnative.to/envoy/configuration/http/http_filters/lua_filter.html) 可以操作,举例：
```
function envoy_on_request(request_handle)
  -- 记录请求信息
  request_handle:logInfo("Authority: "..request_handle:headers():get(":authority"))
  request_handle:logInfo("Method: "..request_handle:headers():get(":method"))
  request_handle:logInfo("Path: "..request_handle:headers():get(":path"))
end

function envoy_on_response(response_handle)
  -- 记录响应状态码
  response_handle:logInfo("Status: "..response_handle:headers():get(":status"))
end
```


