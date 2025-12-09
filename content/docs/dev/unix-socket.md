---
title: "Socket 到底是什么"
weight: 1
date: 2025-12-08
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# Socket 到底是什么

ChatGPT 给出的定义是: 「 **Socket（套接字）** 是应用层与传输层之间的抽象接口，是 Linux 中网络通信的“文件描述符”。
它允许两个进程通过 TCP/UDP 等协议进行通信，既支持本机进程间通信，也支持跨网络通信。」

简单来说, socket 算是一套在应用层与传输层之间的 API, 提供了一种类似读写文件来通信的方式, 包括:
```c
socket()
bind()
listen()
accept()
connect()
send()
recv()
close()
```

应用层根据自己需要的传输层协议 (TCP or UDP) 来调用这一套 API.
```c
socket(AF_INET, SOCK_STREAM, 0); // 创建 TCP socket
socket(AF_INET, SOCK_DGRAM, 0); // 创建 UDP socket
socket(AF_UNIX, SOCK_STREAM, 0); // 创建 Unix Domain socket
```

## Socket 的做与不做

### Socket 会做什么

- 调用协议栈进行连接/监听
  - 例如, 用 socket 创建 TCP 连接时会自动调用内核来处理握手, 监听与挥手.
- 把数据写入协议栈
  - socket 会将要发送的数据与使用的协议发送给内核协议栈来生成对应的报文.
- 从协议栈读取数据
  - 内核协议栈会处理接收到的 TCP / UDP 报文, 并将正文放到内核缓冲区中等待 socket 读取.

### Socket 不会做什么

- 不定义消息格式
  - 具体的消息格式 (JSON / protobuf) 应该由应用层来处理
- 不处理消息边界
  - 这意味着可能会有粘包 / 拆包问题.
  - 需要应用层来处理消息边界.

## 生命周期

以创建 TCP 连接为例, 完整的生命周期如下:

### 服务端

1. 创建 Socket. 这一步会在内核中创建一个 fd 与对应的缓冲区.
   ```c
   int server_fd = socket(AF_INET, SOCK_STREAM, 0);
   ```
2. 绑定地址:
   ```c
   bind(server_fd, (struct sockaddr*)&addr, sizeof(addr));
   ```
3. 启动监听:
   ```c
   listen(server_fd, 1);
   ```
4. 等待连接:
   ```c
   int client_fd = accept(server_fd, NULL, NULL);
   ```
5. 接收 & 发送数据, 注意这一步用的是 `client_fd` 而不是 `server_fd`:
   ```c
   recv(client_fd, buffer, sizeof(buffer) - 1, 0);
   send(client_fd, reply, strlen(reply), 0);
   ```
6. 关闭连接:
   ```c
   close(client_fd);
   close(server_fd);
   ```

### 客户端

1. 创建 Socket.
   ```c
   int sockfd = socket(AF_INET, SOCK_STREAM, 0);
   ```
2. 连接服务端地址:
   ```c
   connect(sockfd, (struct sockaddr*)&addr, sizeof(addr));
   ```
3. 接收 & 发送数据:
   ```c
   recv(sockfd, buffer, sizeof(buffer) - 1, 0);
   send(sockfd, reply, strlen(reply), 0);
   ```
4. 关闭连接:
   ```c
   close(sockfd);
   ```

## epoll 与 socket

epoll 是 Linux 内核提供的 多路复用 I/O 事件通知机制，用于：
- 管理大量 fd（包括 socket）
- 在它们有事件（可读/可写/错误）时一次性返回
- 实现高并发 I/O

在没有 epoll 的时代，如果要处理大量 socket 通信, 可能会：
- 用阻塞 socket 导致大量线程被阻塞
- 用 `select` 或 `poll` 需要每次扫描几千个 fd，效率低, 无法处理高并发

举个简单的用法, 如果我们想处理大量的 socket 通信, 从里面挑一个可以读写的:
```c
// 创建一个 epoll 实例
int epfd = epoll_create(1);

// 以添加单个 socket 为例
int fd = socket(AF_INET, SOCK_STREAM, 0);
struct epoll_event ev;
ev.events = EPOLLIN;   // 当这个 socket 可读的时候通知我
ev.data.fd = fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev); // 注册到 epoll 中

struct epoll_event events[MAX_EVENTS];
while (true) {
    // epoll_wait() 会返回有事件的 socket 数量
    // 并将对应的事件存放在 events 中
    int n = epoll_wait(epfd, events, MAX_EVENTS, -1);
    for (int i = 0; i < n; i++) {
        // ...
    }
}
```
