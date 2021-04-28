---
title: Network IO Models
date: 2021-04-20 22:11:45
tag: NIO
categories: Linux
---
My recent work involved a lot of network IO stuff, like raw TCP/Multicast using Netty and websocket/UDP using Boost.asio. So I went through the NIO models to get a better idea of how it works. I will use TCP server as an example (server need to handle multiple connections). There are lots of charts illustrate these 5 IO models, but I think it is better to understand why we need these models.

First, we need to set up the server socket then make this socket bind with a port and listen. In linux, everything is file descriptor, so does the socket. Below is the Pseudocode.
```
setup fd :
socket fd
bind fd 8080
listen fd
```
##### Block IO
Now if a client asking for connection, our server needs to accept the coming request. So we need to call the `accept` function. However, the `accept` will block the process until recevie a connection request.

After connection request received by server, `accept` will return a new _fd_ which handles the requests from the connected client. `recvfrom` can get data from the client but it will also block the process. And we need our server to handle multiple clients, so we can use `while` for the `accept` to connect new connection and `recvfrom` for data communication. The server process will be like 
```
                       app server
                           |
                |       setup fd       |
                |       while(1)       |
                |     {  accept fd     |   block util connected, return fd1 
                | recvfrom fd1,2... }  |   block util first client send data
                    |      |
Client 1  ----------|      |----------------   Client 2
```
This server can only accept next connection after the first connection sends data because the `recvfrom` block the process. It can not handle more than one client because of this. So we can throw a new thread to handle a connection. So accept only block in main thread waiting for new connection while each new thread for connection will block waiting for receving data.
```
                    app server
                        |
                |     setup fd      |
                |     while(1)      |
                |   {  accept fd    |   block util connected, return fd1 
                |   thorw thread  } | ---------|---------------------|
                |          |                   |                     |
Client 1  ----------|      |             | T1 recvfrom fd1 |       | T2 |
Client 2 ------------------|
```
Each thread will map to one client to handle that connection, and get blocked waiting for data. This model can handle thousand of connections (if we can set up OS to create enough threads). However, it exists some problems:
 - If we have 1000 of clients, the first 999 clients didnt send any data but the 1000th client sent data. In order to get the data from that client, it needs to switch theads 1000 times and 999 switchings are useless and wasting resource.
 - Memory usage waste. Thread stack is independent.

We don't want to throw so many threads and also don't want a blocking function. NIO will be the solution.

##### None Blocking IO
Linux provides a NONBLOCK choice to avoid throwing threads to receive data. So we pass NONBLOCK to socket or use fcntl to set NONBLOCK type fd. When we call recvfrom/accept function if it does receive data/new connection comes, it will return the data/build connection otherwise just return. Here loop all the fds.
```
                        app server
                             |
                |        setup fd         |
                |        while(1)         |
                |      {  accept fd       |
                |   recvfrom fd1,fd2..  } | 
                    |      |
Client 1  ----------|      |
Client 2 ------------------|
```
Now it is possible to use only one thread to handle all connections. It still has some cons:
- Every loop, it will call recvfrom from the linux kernel no matter it has data or not. The complexity is O(n). For N connections, if only one connection has data recevied, it will cause a lot of waste.

So next improvement will be how to reduce the system call(recvfrom) if only one connection receives data? How to ignore the system call which does not have data while looping?

##### Multiplexing
>`select` allow a program to monitor multiple file descriptors, waiting until one or more of the file descriptors become "ready" for some class of IO operation.

The while loop will be like:
1. use `FD_SET` to pass the socket fd(listen) in to fd array.
2. check the return fd array has the listen fd or not
3. if it has listen fd, call accept to build a new connection and get a connection fd.
4. pass the new connection fd into the fd array
5. check fd array has the connection fd or not
6. if it has connection fd, call recvfrom
```
                        app server
                             |
                |        setup fd         |
                |        while(1)         |
                |   {  select(fd*) -> num |
                |         accept(fd)      |
                |        recvfrom(fd*)  } | 
                    |      |
Client 1  ----------|      |
Client 2 ------------------|
```
Everytime loop the fds, the complexity is O(1) and O(n) for recvfrom.

- Every loop, application needs to pass all file descriptors into select.
- Kernel needs to loop all the fds, for kernel it is O(N)

We have a `epoll` function to solve this.
First, `epoll_create` will create a space in kernel then `epoll_ctl` can add or delete listen fd or read/write fd in that space. If the fd is ready, kernel will pass that fd to another space. We can use epoll_wait to get ready fds. It should look like the following chart.
```
                        app server
           |---------------- |-------------------------|
           |                 |epoll_ctl(add/del)       |
           |epoll_create     |                         |epoll_wait
           |                 |                         |
           |    |-------------------|                  |
           |    |     listen fd     |        |-----------------|
    |---------- |   read/write fd   |--------|       fd        |---------|
    |           |-------------------|        |-----------------|        |
    |               kernel                                              |
    |-------------------------------------------------------------------|
```
```
int listenfd = socket();
bind(listenfd, addr);
listen(listenfd);
int efd = epoll_create(0);
epoll_ctl(efd, epoll_ctrl_add, listenfd, &ev);
while(true) {
    epoll_event evs[size];
    int nevent = epoll_wait(efd, evs, size, timeout);
    for(int i; i<nevent; i++) {
        if(e->fd==listenfd) {
            int fd = accept(listenfd);
            epoll_ctl(efd, epoll_ctrl_add, fd, &ev);
        }
        else {
            read/write/logic
        }
    }
}
```
Application use multiplxing to get ready fds (accpet/recvfrom/write) but the app needs to do the operation itself. It is the synchronize model (BIO, NIO, Multiplexing). Linux is hard to achieve AIO (asynchronize IO) considering OS safety (Windows use Iocp as AIO).

##### Signal-driven
##### AIO
These two are not common used.