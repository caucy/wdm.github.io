# 三次握手阶段sync重传问题排查
## 现象
客户报障，线上偶发超时，通过抓包发现线上LB 存在三次握手阶段，syn 后一直不回ack引发重传的情况。

## tcp 的半连接和全连接队列
![syn](https://github.com/user-attachments/assets/96209a74-b369-45e7-8586-cfe0db77b9ad)

通过ss 可以看到半连接队列和全连接队列大小:
```
-l listen
-n 不解析服务名
-t tcp

ss -lnt
```
Recv-Q 表示半连接队列
Send-Q 表示全连接队列

通过 netstat -s 命令可以查看 TCP 半连接队列、全连接队列的溢出情况
```
netstat -s |grep -i listen
    911 times the listen queue of a socket overflowed
    911 SYNs to LISTEN sockets dropped
```

## 如何扩大半全连接队列
TCP 全连接队列足最大值取决于somaxconn 和backlog 之间的最小值，也就是min(somaxconn, backlog)
全连接跟somaxconn 和backlog 有关系，内核有个比较复杂的算法

```
sudo sysctl -w net.core.somaxconn=210000
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=210000
sudo sysctl -w net.ipv4.tcp_syncookies=1
```

## 还有哪些也需要调整
1. nginx 最大fd 限制 work_rlimit_nofile = 65535, 可以设置一个比较大的值100w
2. 其他内核参数
 ```
 fs.inotify.max_user_instances: 100w
 fs.inotify.max_user_watches: 100w
 net.core.somaxconn: 20w
 net.core.netdev_max_backlog: 20w
 net.ipv4.tcp_max_syn_backlog:20w

 ```

## 参考资料

实验参考：https://xiaolincoding.com/network/3_tcp/tcp_queue.html#%E5%AE%9E%E6%88%98-tcp-%E5%85%A8%E8%BF%9E%E6%8E%A5%E9%98%9F%E5%88%97%E6%BA%A2%E5%87%BA
