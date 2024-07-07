# envoy 请求的流转过程

## 连接accept过程

### work 线程的启动
```
MainCommonBase::run()
  -- server_->run()
    --InstanceImpl::run()
      --InstanceImpl::startWorkers()
        --addListenerToWorker()//将listener 同线程绑定
        --WorkerImpl::start()
            -- handler_->addListener()--dispatcher_->run()//等待事件的触发
```
### 监听器的加载
tcp 和 dup 对应不同的处理流程，以tcp 举例，需要设置accept 对应的回调。
在tcp_listener_impl.cc文件中，initializeFileEvent方法注册accept回调方法onSocketEvent，
其中参数中的Event::FileTriggerType::Level表示边沿触发模式，减少网络事件的通知数量，bind_to_port表示是否真正启动网络监听。

```
TcpListenerImpl::TcpListenerImpl(Event::DispatcherImpl& dispatcher, Random::RandomGenerator& random,
                                 SocketSharedPtr socket, TcpListenerCallbacks& cb,
                                 bool bind_to_port, bool ignore_global_conn_limit)
    : BaseListenerImpl(dispatcher, std::move(socket)), cb_(cb), random_(random),
      bind_to_port_(bind_to_port), reject_fraction_(0.0),
      ignore_global_conn_limit_(ignore_global_conn_limit) {
  if (bind_to_port) {
    socket_->ioHandle().initializeFileEvent(
        dispatcher, [this](uint32_t events) -> void { onSocketEvent(events); },
        Event::FileTriggerType::Level, Event::FileReadyType::Read);
  }
}
```
### accept 的回调
**onSocketEvent onSocketEvent方法执行ActiveTcpListener::onAccept回调方法进入新连接接收阶段。**
```
void TcpListenerImpl::onSocketEvent(short flags) {

    const Address::InstanceConstSharedPtr& local_address =
        local_address_ ? local_address_ : io_handle->localAddress();

    const Address::InstanceConstSharedPtr remote_address =
        (remote_addr.ss_family == AF_UNIX)
            ? io_handle->peerAddress()
            : Address::addressFromSockAddrOrThrow(remote_addr, remote_addr_len,
                                                  local_address->ip()->version() ==
                                                      Address::IpVersion::v6);

    cb_.onAccept(
        std::make_unique<AcceptedSocketImpl>(std::move(io_handle), local_address, remote_address));
  }
}
```

在active_tcp_listener.cc文件中，对新连接调用createListenerFilterChain方法，该方法根据监听器的配置config_创建监听过滤器filterChain，
并调用continueFilterChain方法执行监听过滤器filterChain内的各个回调方法，处理连接建立过程
```
void onSocketAccepted(std::unique_ptr<ActiveTcpSocket> active_socket) {
    // Create and run the filters
    config_->filterChainFactory().createListenerFilterChain(*active_socket);
    active_socket->continueFilterChain(true);
...
  }


void ActiveTcpSocket::newConnection() {
  // Check if the socket may need to be redirected to another listener.
  Network::BalancedConnectionHandlerOptRef new_listener;

  if (hand_off_restored_destination_connections_ &&
      socket_->connectionInfoProvider().localAddressRestored()) {
    // Find a listener associated with the original destination address.
    new_listener =
        listener_.getBalancedHandlerByAddress(*socket_->connectionInfoProvider().localAddress());
  }
  if (new_listener.has_value()) {
    // Hands off connections redirected by iptables to the listener associated with the
    // original destination address. Pass 'hand_off_restored_destination_connections' as false to
    // prevent further redirection.
    // Leave the new listener to decide whether to execute re-balance.
    // Note also that we must account for the number of connections properly across both listeners.
    // TODO(mattklein123): See note in ~ActiveTcpSocket() related to making this accounting better.
    listener_.decNumConnections();
    new_listener.value().get().onAcceptWorker(std::move(socket_), false, false);
  } else {
...
// Create a new connection on this listener.
    listener_.newConnection(std::move(socket_), std::move(stream_info_));
}
}

```

**newConnection方法将真正创建连接的对象, 并与filter 绑定**

```
// Find matching filter chain.
  const auto filter_chain = config_->filterChainManager().findFilterChain(*socket);
  if (filter_chain == nullptr) {
    // 若未匹配到L4过滤器，则断开连接
  }
  ...
  // （1）建立跟下游的连接（更准确的描述是跟对端的连接）
  auto server_conn_ptr = dispatcher().createServerConnection(
      std::move(socket), std::move(transport_socket), *stream_info);
  // （2）创建四层网络过滤器
  const bool empty_filter_chain = !config_->filterChainFactory().createNetworkFilterChain(
      *server_conn_ptr, filter_chain->networkFilterFactories());
  // （3）将连接跟监听器关联起来，如果监听器配置发生变化，为了保证现有的连接的过滤器同步更新，会重建过滤器。
  newActiveConnection(*filter_chain, std::move(server_conn_ptr), std::move(stream_info));
```
### 注册read、write handler
createServerConnection 会注册链接的读写回调为**onFileEvent**
```
ConnectionImpl::ConnectionImpl(Event::Dispatcher& dispatcher, ConnectionSocketPtr&& socket,
                               TransportSocketPtr&& transport_socket,
                               StreamInfo::StreamInfo& stream_info, bool connected)
    : ConnectionImplBase(dispatcher, next_global_id_++),
      write_buffer_(dispatcher.getWatermarkFactory().createBuffer(
          [this]() -> void { this->onWriteBufferLowWatermark(); },
          [this]() -> void { this->onWriteBufferHighWatermark(); },
          []() -> void { /* TODO(adisuissa): Handle overflow watermark */ })),
      read_buffer_(dispatcher.getWatermarkFactory().createBuffer(
          [this]() -> void { this->onReadBufferLowWatermark(); },
          [this]() -> void { this->onReadBufferHighWatermark(); },
          []() -> void { /* TODO(adisuissa): Handle overflow watermark */ })),
  // We never ask for both early close and read at the same time. If we are reading, we want to
  // consume all available data.
  socket_->ioHandle().initializeFileEvent(
      dispatcher_, [this](uint32_t events) -> void { onFileEvent(events); }, trigger,
      Event::FileReadyType::Read | Event::FileReadyType::Write);
}
```

## http连接读写流程
**onFileEvent** 是request connection 的读写入口


