# envoy 如何实现http协议解析

envoy 需要一个状态机解析http 协议， http_parser 是c 实现一个高性能http parser, 所以envoy 需要维护一个parser, 并在收到data 后循环调用http_parser_execute

http_parser 的使用例子：

```
// parser 初始化
http_parser_init(&parser, type);
memset(&messages, 0, sizeof messages);

// 注册每个阶段的回调
static http_parser_settings settings =
  {.on_message_begin = message_begin_cb
  ,.on_header_field = header_field_cb
  ,.on_header_value = header_value_cb
  ,.on_url = request_url_cb
  ,.on_status = response_status_cb
  ,.on_body = body_cb
  ,.on_headers_complete = headers_complete_cb
  ,.on_message_complete = message_complete_cb
  ,.on_chunk_header = chunk_header_cb
  ,.on_chunk_complete = chunk_complete_cb
  };

// 当libevent 来数据后调用http_parser_execute
nparsed = http_parser_execute(parser, &settings, buf, recved);

```
