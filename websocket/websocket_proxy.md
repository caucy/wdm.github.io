# websocket跑私有协议
* grpc 协议能跑在websocket 上吗？

* 如果要维护长链接选grpc 还是websocket,还是其他协议？

* websocket 是http 协议吗？跟http 协议有什么关系？ 


因为7层代理比较成熟，而且一般动态加速，7层比4层动态加速要便宜，所以一般长链接服务都倾向于跑在websocket 上，有很多公司在websocket
上跑私有协议，当然也包括grpc。

## grpc如何实现跑在websocket 协议上？
