#Muduo源码阅读（3）：Muduo中的Reactor模式
muduo是一个以Reactor模式为核心的网络库，所以Reactor模式的各个组件在muduo中都有对应的部分。虽然在具体实现和理念上muduo有它自身的特点，但其内在的组织结构是高度符合Reactor模式的。可以说理解了Reactor模式，就会对muduo的设计有比较清楚的认识，muduo的相关介绍请见[1]及作者的相关博文。下图是muduo的类图，各个模块与Reactor模式中各组件的对应关系是：
1. Initialization dispatcher —— EventLoop
2. Synchronous Event Demultiplexer —— Poller
3. Handles —— FileDescriptor、Channel
4. Event Handler —— TcpConnection、Acceptor、Connector

》》》》》》》》》》》》》》》》》》》》》》》

补充muduo的UML图

》》》》》》》》》》》》》》》》》》》》》》》

上述的对应关系并不完全严格，但从功能上有这样大体的对应关系。其实仅仅从这两幅图的结构上，大家也可以看到一定的相似之处吧。

从这幅类图上我们可以看到TcpConnection、Acceptor、Connector这三个类。这三个类的功能从它们的名称上就可以大致看出来：
* TcpConnection：封装一次Tcp连接的过程，同时我们可以在这个类中指定对连接建立后对服务的处理过程。
* Acceptor：用于初始化服务端的socket套接字，包括监听、绑定等，完成对新连接的accept。
* Connector： 封装客户端的主动连接过程。`FIXME`
 
这三个类实现的是客户端/服务器编程中典型的功能，但为什么要将功能划分成这样的三个类实际上也是有章可循的。

在说明这种设计背后的原因时，先让我们考虑另外一种设计方式。我们知道在面向连接的协议中有两种角色，即主动请求连接的角色和被动接受连接的角色。对应到TCP协议中，就是客户端和服务端。对于套接字编程来说，二者都需要建立套接字并绑定到某个地址上。我们可以只定义一个类Socket用于同时描述客户端及服务器的套接字，使用时仅仅通过调用不同的构造函数及接口函数来区分二者。考虑下面的类定义：
```
class Socket 
{
public:
    // 创建监听套接字
    Socket(unsigned int port);
    // 创建客户端连接套接字
    Socket(const char *pHostname, unsigned int port);
    int bind_listen();
    int send(...);
    int recv(...);
    int accept(...);
private:
    int _fd;
    struct sockaddr_in _sockAddress;
}
```
有了上述的定义后，服务端的伪代码可以这样编写：
```
Socket sock(port);
sock.bind_listen();
while (true) 
{
    sock.accept();
    server_process();
}
```
客户端可以有如下结构:
```
Socket sock(pHostname, port);
sock.connect();
client_process();
```
但这种对客户端、服务器不加区分的设计并不好，毕竟二者具有不同的初始化过程，而且server_process()和client_process()实际上是和连接的初始化过程无关的。很明显，上述的设计将客户端初始化、服务端初始化、服务处理这三部分混杂在了一起。要优化这样的一个设计，很直接的解决思想就是解耦，关键的问题是这“耦”如何去解呢？

ACE网络库的作者Schmit针对这个问题提出了一种`Acceptor and Connector`的模式。在他的论文[2]中指出通过这种模式，可以将通信协议中的被动、主动这两种初始化角色和连接建立后要完成的任务解耦。这个模式的提出是基于连接的建立和初始化过程与连接建立完成后进行的服务无关。客户端连接的主动初始化、服务端连接的被动初始化、服务的处理都可以视为事件的到来（`这里怎么描述FIXME`），所以回想Reactor模式，我们自然可以将这三种任务通过Event Handler实现，当特定事件到来时触发这三种任务的处理。`FIXME:以下详述这三个模块的处理机制`

如果我们看Muduo中Acceptor、Connector、TcpConnection的源代码，会发现这三个类有以下几个共同点：
1. 包含Channel类型的成员变量，也是除了EventLoop之外，Muduo中仅有的包含Channel的类型。
2. 有类似handleRead()、handleWrite()的私有函数。
3. 有回调函数作为成员变量，并且有设置该回调函数的接口。

有这几个共同点并非偶然，它们与Event Handler的实现关系非常密切。具体的实现过程是这样的：这三个类中的回调函数成员变量会在handle\*()类型的函数中调用，用于处理读写等功能； 而handle\*()函数会被绑定到Channel成员变量channel_中，channel_实际上就有了特定类型的钩子函数。通过Reactor机制，会在特定类型事件发生时调用相应的钩子函数，从而Acceptor、Connector、TcpConnection都可以完成相应的功能。

这里的实现方式是由Muduo很有意思的回调函数机制决定的，我们在下面在介绍这方面的内容。在schmit的原始论文中用的是继承方式，Acceptor、Connector以及ServiceHandler都继承EventHandler接口，这里不展开说了，有兴趣的可以看下他的论文[2]。

可以看到这样的一种“模式”相对上文举的将几个部分混杂一起的例子要好很多。我们完全可以根据需要编写函数指定被动、主动方如何初始化连接，连接完成后进行怎样的服务。而且服务处理与连接建立之间基本没有耦合，在保证模块间结构的同时也有很好的灵活性。

[1] http://blog.csdn.net/solstice/article/details/5848547

[2] [Acceptor and Connector by DC Schmit](www.cs.wustl.edu/~schmidt/PDF/Acc-Con.pdf‎)

