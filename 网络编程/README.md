### NetWork Programing
#### Unix域套接字和TCP/IP套接字  


#### Reactor随想  
这里来讲讲`Reactor`这个网络编程设计模式的起源，它是基于`IO Multiplexing`和`Non-Blocking fd`的框架。  
我们先来看一下`IO multiplexing`和`Non-Blocking fd`的网络编程样子吧：  
对于Server端：  
```python  
create socket    
set socket options    
bind IP:PORT  
listen   
while True:   
{   
  IO multiplexing system call: poll epoll select  // will Block here    
  for activeFd, event in events     
    if activeFd is Listen_socket   
      handleConnection()   
    else if activeFd is othersFd && event & POLLIN   
      handleRead()    
    else if activeFd is othersFd && event & POLLOUT    
      handleWrite()
} 
```
在网络编程中有很多是Routine的，比如服务器端需要先1）创建套接字 2）绑定IP：PORT 3）进行监听 4）处理已连接socket等等，这一套流程是刻板的以至于长期写网络编程的同学会对这套流程非常熟悉，熟悉到厌烦的地步，于是就开始抽象出公有的不变量，各类框架也应运而生。  
我们如果能够控制网络编程中几个重要的节点的逻辑，将这部分逻辑注册到某个网络的框架中，以回调的方式运行我们的代码。  
在陈硕的《Linux多线程服务端编程》中，他提到了TCP网络编程最本质的处理三个半事件，即  
* 连接的建立：服务端`accept`的时候和客户端成功发起`connect`的时候  
* 连接的断开：包括主动断开和被动断开（`read`返回0）  
* 消息到达：文件描述符可读，这是最为重要的一个事件，对它的处理方式决定了网络编程的风格（阻塞还是非阻塞，如何处理分包，应用层的缓冲如何设计）  
* 消息发送完毕：对于低流量的服务，这个事件不必关心，而且这里需要强调的是发送完毕不是指对方已经收到，而是仅仅将数据从用户空间缓冲区写入到内核缓冲区，当流量很大的时候，内核缓冲区会被塞满。  
我们只需要在这些事件发生时刻注册回调函数即可。  
当然，像刚刚写的伪代码里将并发策略和业务处理逻辑放在同一个线程中，随着业务逻辑的复杂，并发度会下降，这也是陈硕改进Reactor模式的原因之一，采取的方法是：Reactors + thread pool，Reactors 用来保证处理连接的响应，thread pool将网络IO和业务逻辑分离。既可以适应高并发的场景，也可以适应计算密集的场景。
