High performance websocket push component
ws push optimization
Implemented these optimizations.

1. When there are more writes and fewer reads, use epoll io multiplexing to avoid creating a read coroutine.

2. Shard storage connections to reduce lock granularity and increase push concurrency.

3. Use cache when writing, and write in batches at a fixed time to reduce system calls and coroutine scheduling.



1、epoll
1. Implementation reference, https://github.com/eranyanay/1m-go-websockets is only applicable when there are more writes and fewer reads, and the time-consuming operations after reading should not be too frequent.

When reading here, we need to consider the situation of half message. https://studygolang.com/topics/13377 When the websocket data packet is received incompletely, the go net official library will block when it is unreadable. If we block here, we have to wait until he finishes reading before we can process other things.

Solution : If the message is not fully read, cache it and the starting position of the read will fall back to the position where the complete data was read last time. Here you need to DIY bufio.Reader to cache the messages that have not been read.

2. Use write cache
If there are a lot of small packets, sending them directly will result in many system calls, and each system call will cause a scheduling and consume CPU. If the real-time requirement is not high, we can write them in batches at a fixed time.

Thanks to the very flexible usage provided by github.com/gobwas/ws, we can temporarily convert the data we want to write into websocket packets and cache them, and write them in batches at a scheduled time. In addition, encoding websocket packets in advance can also avoid encoding websocket packets again when sending each connection, which improves efficiency.

3. Shard storage connection
Mainly refer to https://github.com/owenliang/go-push

The main idea is to shard storage connections to reduce the granularity of locks and the concurrency of pushes.

Compile and run
This program depends on etcd, so you need to build the etcd service in advance and configure it in the configuration file.

Compile make build or compile using dockermake dockerBuild

Run make runor docker runmake dockerRun

Implementing functions
Push data to public channels

Push data to all connections

Push data to a private channel

Stay connected
A ping string needs to be sent within a specified time. Since the browser does not support sending PING data frames, all operations for sending ping strings are of Text type.

Subscription Operation
The basic structure of client request data

Request address: ws://192.168.2.99:9992/ws

{
    "code":1,
    "topic":"test",
    "data":"any"
}
code: indicates the type of operation

1Subscribe to public channels.
2Unsubscribe from public channels.
3. Login operation requires you to implement the login logic yourself.
4. Logout operation requires you to implement the logout logic yourself, which will exit all private channels you subscribed to.
5To subscribe to a private channel, you need to log in.
6Unsubscribe from private channels. Log in is required.
topic: the name of the channel you subscribed to

data: Any type, reserved field. If it is a login operation, you can use this field to pass your login credentials.

Background push
Background push provides http interface and grpc interface

http interface : provided by grpc-gateway

POST /v1/pushData

JSON format data

{
    "uid":"123456",
    "topic":"test",
    "data":"5pWw5o2u"
}
uid : user id. When pushing to a private channel, specify the user's id. Otherwise, it is empty.

topic : The name of the channel. Leave it empty if you want to push to everyone.

data: The specific content of the push. The pushed data must be base64 encoded.

grpc interfacegithub.com/luxun9527/gpush/proto

protobuf format

message Data{
  string uid =1;
  string topic=2;
  bytes data=3;
}