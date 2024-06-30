## envoy 有哪些filter？
envoy 没有一个命令能dump 所有支持的filter，不是很友好，如果想支持某个版本的envoy 支持哪些filter，可以看源码source/extensions/extensions_build_config.bzl，里面定义了所有的filter信息。

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
