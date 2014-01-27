#Muduo源码阅读（7）：Event Handler之一 —— Acceptor类
在前面的几篇源码阅读中我们先后介绍了Reactor模式中的Dispatcher、Demultiplexer以及Handler，接下来开始介绍Muduo中的Event Handler。Event Handler定义了一系列对特定事件处理的钩子函数，不同的Event Handler根据其特定的功能的不同其钩子函数也不同。Muduo中的Event Handler有Acceptor、Connector、TcpConnection、TimerQueue，本文先讨论Acceptor。
##1 Acceptor设计模式及处作用过程
Acceptor从名字上来看与套接字编程中服务器端的accept函数很接近，实际上也有很多的关联。Muduo中的Acceptor封装了服务器端被动建立连接所需的套接字定义、绑定、监听、accept等过程。将Acceptor作为Event Handler实际上是一种称为`Acceptor`的设计模式[1],提出者同样为提出`Reactor`模式的*D.C.Schimit*。Acceptor起到了将被动连接的建立与服务处理过程解耦的作用，主要完成的任务按顺序有以下两点：
1. 被动连接端的初始化，其实就是套接字的相关初始化操作
2. 客户端连接请求的处理，通过Reactor的事件分发机制完成
Acceptor之所以可以被设置为Event Handler，是由于客户端的连接请求对Acceptor来说是读事件，所以当Acceptor的Channel可读时就会调用相应的读回调函数完成连接的建立。Acceptor并没有预设连接建立方式，具体的连接建立由newConnectionCallBack_决定，也就是说Acceptor与传输层的连接初始化协议无关，并不局限于Tcpl协议。
##2 Acceptor套接字初始化相关代码
套接字初始化首先是套接字的创建和绑定，这部分在Acceptor的构造函数中完成。
```
Acceptor::Acceptor(EventLoop* loop, const InetAddress& listenAddr)
  : acceptSocket_(sockets::createNonblockingOrDie()), //创建非阻塞套接字
    acceptChannel_(loop, acceptSocket_.fd()),         //利用套接字创建channel对象
{
  ...
  acceptSocket_.bindAddress(listenAddr);    // 将acceptSocket_绑定到监听地址上
  // 将acceptChannel_的读回调函数设置为Acceptor::handleRead
  acceptChannel_.setReadCallback(    
      boost::bind(&Acceptor::handleRead, this));
  ...
}
```
接下来就是套接字监听，这步完成后套接字转变成监听套接字，等待客户端连接请求的到来。
```
void Acceptor::listen()
{
  // 调用套接字函数listen()
  acceptSocket_.listen();
  // 设置当与Acceptor相关的Channel有读事件到来时进行通知，
  // 并将Channel插入Demultiplexer
  acceptChannel_.enableReading();
}

```
##3 Acceptor的读事件处理
对监听套接字来说，当有连接请求到来时套接字变成可读状态，所以连接请求到来是Acceptor的一个读事件。又由于Acceptor是一个Event Handler，所以连接请求到来的处理和一般的Reactor模式处理方式相同。具体的说，就是客户端主动连接到来时会被Demultiplexer捕获，EventLoop调用与Acceptor相关Channel的*handleEvent()*函数，由于在这个Channel上发生的是读事件，调用相应读回调函数，也就是*Acceptor::handleRead()*。在*Acceptor::handleRead()*中首先完成新连接的accept，之后
调用连接建立回调函数newConnectionCallback_完成建立连接。newConnectionCallback_由外界传入，所以连接建立的过程可以由客户定义。
```
void Acceptor::handleRead()
{
  InetAddress peerAddr(0);
  int connfd = acceptSocket_.accept(&peerAddr); //从这个地址开始接受信息，并创建新的套接字connfd
  ...
  newConnectionCallback_(connfd, peerAddr);  //调用连接回调函数，完成接收
  ...
}

``` 

[1] Acceptor
