# websocket跑私有协议
* grpc 协议能跑在websocket 上吗？

* 如果要维护长链接选grpc 还是websocket,还是其他协议？

* websocket 是http 协议吗？跟http 协议有什么关系？ 


因为7层代理比较成熟，而且一般动态加速，7层比4层动态加速要便宜，所以一般长链接服务都倾向于跑在websocket 上，有很多公司在websocket
上跑私有协议，当然也包括grpc。

## 1. grpc如何实现跑在websocket 协议上？
其实websocket协议握手完成后，就是一个纯4层代理，上面跑啥协议都可以。

下面这是websocket client 的一个[demo](https://github.com/gorilla/websocket/blob/master/examples/chat/client.go):
```
// serveWs handles websocket requests from the peer.
func serveWs(hub *Hub, w http.ResponseWriter, r *http.Request) {
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		log.Println(err)
		return
	}
	client := &Client{hub: hub, conn: conn, send: make(chan []byte, 256)}
  // func (c *Conn) UnderlyingConn() net.Conn, 可以通过UnderlyingConn 拿到底层的net>conn 
	client.hub.register <- client

	// Allow collection of memory referenced by the caller by doing all work in
	// new goroutines.
	go client.writePump()
	go client.readPump()
}
```
可以通过UnderlyingConn 拿到底层的net>conn ，传给grpc dail 即可让协议跑在grpc 上

## 2. grpc 传入websocket 接口并做协议封装
grpc WithContextDialer 可以传递一个回调函数，返回实现net.Conn 接口的函数出来。这个connectWs 实现比较取巧，可以直接走net.Conn 收发数据，也可以封装一个结构体，
走websocket.Conn SendMessage ReciveMessage。这里可以做多种协议的拆解。

```	
	//wsDial 是websocket 的封装
	wsDial := grpc.WithContextDialer(func(ctx context.Context, target string) (net.Conn, error) {
		return connectWs(ctx, "ws://xxxx:xxx")
	})
	var conn *grpc.ClientConn
	conn, err := grpc.Dial(":xxx", grpc.WithInsecure(), wsDial)
```

websocket server 握手结束后，按相同封装解封装即可。

