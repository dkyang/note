#reactor模式

@(muduo)[源代码阅读 草稿 设计模式]

Channel成员函数，设置感兴趣事件类型
```
void enableReading() { events_ |= kReadEvent; update(); }
void disableReading() { events_ &= ~kReadEvent; update(); }
void enableWriting() { events_ |= kWriteEvent; update(); }
void disableWriting() { events_ &= ~kWriteEvent; update(); }
```
当注册一个Concrete Event Handler到Initiation Dispatcher，要告知这个Handler感兴趣的事件类型。


异步事件demultiplexer-select（EventLoop）
handler/event handler -Channel


客户端连接到日志服务：
总结为下面的步骤。
1. 注册Logging Acceptor到Initiation Dispatcher，等待处理连接请求；
2. Initiation Dispatcher的handle_events方法被调用；
3. Initiation Dispatcher调用Synchronous Demultiplexer的select方法，等待连接请求或日志数据的到达；
4. 一个客户端连接到日志服务器；
5. Initiation Dispatcher通知Loging Acceptor：有新的连接请求到达；
6. Logging Acceptor接受新的请求；
7. Logging Acceptor创建一个新的Logging handler去为新的客户服务；
8. Logging Handler把handle注册到Initiation Dispatcher。

客户端连接到Tcp服务器的过程（模仿上面的步骤）
。。。。。。。。。。。。。。。。。

？Initiation Disaptcher每个线程一个？


Acceptor-Connector模式？分离了服务的实现和服务的激活

event-driven programming : reactor(non-blocking), procator(asynchrons)


reactor pattern + thread pool + non-blocking

Each Handler is associated with a SocketChannel and the Selector maintained by the Reactor class. 
reactor论文阅读记录
1. 通过多线程处理并发（“thread-per-connection）


缺陷有：
* 效率——上下文切换
* 当服务变更时解复用器代码的变化
* OS移植

2.demultiplexing and dispatching 

实现新的应用服务不应该影响通用的解复用及分派机制
解决方法：
Integrate the synchronous demultiplexing of events and the
dispatching of their corresponding event handlers that pro-
cess the events.
In addition, decouple the application-
specific dispatching and implementation of services from
the general-purpose event demultiplexing and dispatching
mechanisms.


* handles: 由句柄管理的资源
* 同步事件解复用器：select、poll、epoll等
* Initiation Dispatcher：注册、移除、分派event handler的接口
* Event Handler：实现hook method的接口，根据特定服务不同实现
对于日志服务器而言，有logging handler、acceptor handler这两种event handler

EventLoop，这其实是个 Reactor，用于注册和分发 IO 事件。Muduo 遵循 one loop per thread 模型，多个服务端(TcpServer)和客户端(TcpClient)可以共享同一个 EventLoop，也可以分配到多个 EventLoop 上以发挥多核多线程的好处。这里我们把五个服务端用同一个 EventLoop 跑起来，程序还是单线程的，功能却强大了很多




***************************************************************
muduo与reactor模式
# reactor模式的意义及产生背景
Reactor模式经常被翻译为“反应器”模式，但我认为这个翻译并不夠直观。其实说白了，所谓react就是对外界事件到来的一种被动的处理。比如在零下的室外环境中，我们会情不自禁的起鸡皮疙瘩，这就是我们对“感到寒冷”这个事件的反应；再比如我们因得知自己中了六合彩而高兴的手舞足蹈，这就是对“中大奖”这个事件的反应；对于软件而言，我们点击了界面上的某个按钮会弹出对话框，弹出对话框当然是对“点击按钮”事件的反应。可以看到，上述几种情形中都不是主动的发出动作，而是因为事件的到来而触发了反应的产生。我们知道，软件的其实是一种对现实生活现象建模，再通过计算机完成求解的过程，而reactor模式就是针对上面这种现象而提出的。

现在让我们用较专门的语言再来阐述reactor模式。首先联想一下普通函数调用的机制：程序调用某函数，函数执行，程序等待，函数将结果和控制权返回给程序，程序继续处理[1]。这是一种顺序的处理方式，但reactor并不主动会调用函数，而是首先将相应的处理函数注册到reactor上，当我们感兴趣的事件发生到来再通知reactor调用之前注册的函数完成处理，这些函数就是所谓的“回调函数”，而到来的事件可以多种多样，比如信号、I/O读写、定时、GUI消息等。

回到网络编程，我们处理的其实就是各种各样的事件。服务器接收到客户端的连接、读数据、写数据、接收到错误，这些都是事件。一般的处理都是一种主动调用函数的方式，比如最简单的TCP服务器模型：
```
socket(...);
bind(...);
listen(...);
for ( ; ; ) {
    accept(...);
    if ((childpid = fork()) == 0) {
        ....
    }
}
```
服务器端在完成基本的套接字建立、绑定、监听后，在for循环中主动的调用accept等待客户端连接到来，如果没有新连接，会阻塞在accept上，如果有新连接，则fork出一个子进程用以完成连接的处理。我们知道，随着并发连接的增多，我们需要fork出很多的子进程，当然也有其他的客户/服务器编程范式[2]，比如每一个客户通过一个线程处理。但类似的方式都会带来各种问题[3],比如效率、编程复杂度、移植性等，具体可参考[5]。而reactor模式在高并发的场景下就有了优势。reactor模式一般与非阻塞I/O结合，在处理一个事件时不用等待事件的完成，而是转而去处理其他的任务，当这个事件处理完毕后再去通知reactor，从而用单个线程可以完成多线程才能完成的任务，提高了系统的吞吐量。[4]中用一个餐馆的例子解释了reactor模式在高并发下的优势，很有意思。具体来说，reactor模式有如下优点[1,3]：
* 响应快，不必为单个同步时间所阻塞，虽然Reactor本身依然是同步的；
* 编程相对简单，可以最大程度的避免复杂的多线程及同步问题，并且避免了多线程/进程的切换开销；
* 可扩展性，可以方便的通过增加Reactor实例个数来充分利用CPU资源；
* 可复用性，reactor框架本身与具体事件处理逻辑无关，具有很高的复用性；

# Reactor模式的结构 
要实现Reactor模式，主要需要四个部分的组件[3]：
* Initialization dispatcher
* Synchronous Event Demultiplexer
* Handles
* Event Handler

其中的Handles用于标识通过操作系统管理的各种资源，包括套接字、打开文件、锁、定时器等，在unix系统中是文件描述符，在windows系统中是句柄。当有事件在Handles上发生时，我们可以通过Synchronous Event Demultiplexer得知。Synchronous Event Demultiplexer就是解复用的系统，比如我们熟知的select、poll、epoll等。Initialization dispatcher实现了事件的注册、删除、分发的接口，是用于驱动的主模块。Synchronous Event Demultiplexer也是它的一个组件，用于等待新事件的发生。当事件发生时，Demultiplexer通知dispatcher，dispatcher再根据事件的类型使用相应的Event Handler完成事件的处理。所以Event Handler是最终用于实际处理事件的组件，一般来说会根据事件类型的不同实现各种类型的钩子函数。下图就是Reactor模式的结构：

》》》》》》》》》》》》》》》》》》》》》》》》》》》》》》》》》》》》

（待添加图片！！！！！！！！）

》》》》》》》》》》》》》》》》》》》》》》》》》》》》》》》》》》》》

具体的处理过程如下：
1. 一个应用将某个Event Handler注册到Initialization dispatcher上，并且指定其感兴趣的事件，希望当有感兴趣的事件在关联的handle上发生时去通知这个Event Handler。
2. 当Event Handler注册完毕后，Initialization dispatcher在一个事件循环中通过Synchronous Event Demultiplexer等待事件的发生。比如一个监听套接字等待accept一个新的客户端连接。
3. 当Handle关联的资源就绪时，Synchronous Event Demultiplexer通知Initialization dispatcher，比如当一个TCP套接字可读时。
4. 当某个Handle上有特定事件发生时，Initialization dispatcher找到和这个Handle关联的Event Handler，根据事件的类型调用Event Handler相应的钩子函数完成事件处理。

#muduo与Reactor模式
muduo是一个以Reactor模式为核心的网络库，所以Reactor模式的各个组件在muduo中都有对应的部分。虽然在具体实现和理念上muduo有它自身的特点，但其内在的组织结构是高度符合Reactor模式的。可以说理解了Reactor模式，就会对muduo的设计有比较清楚的认识，muduo的相关介绍请见[6]及作者的相关博文。下图是muduo的类图，各个模块与Reactor模式中各组件的对应关系是：
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
* Connector： 封装客户端的主动连接过程。？？？？？？？？？？？？？？？
 
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

ACE网络库的作者针对这个问题提出了一种Acceptor and Connector的模式。在他的论文[7]中指出通过这种模式，可以将通信协议中的被动、主动这两种初始化角色和连接建立后要完成的任务解耦。这个模式的提出是基于分布式协议中端点之间消息的交换任务和以下三点独立：
1. 消息交换的两个端点中，哪个是连接的主动请求方，哪个是连接的被动接收方。连接正式建立后，数据的交换并不需要这个信息。
2. 网络编程接口和建立连接内部使用的协议。
3. 创建、连接服务的方式以及并发策略。

！！！！！！！！！！！！！！这部分重写！！！！！！！！！！！

其实这也就是说连接的初始化过程和连接建立后完成的任务独立，所以我们当然要将连接的初始化与连接建立后的处理分开。客户端连接的主动初始化、服务端连接的被动初始化、服务的处理都可以视为事件的到来（这里怎么描述更好？？？），所以回想Reactor模式，我们自然可以将这三种任务实现为Event Handler，当特定事件到来时触发这三种任务的处理。

如果我们看这三个类的源代码，会发现都包含了Channel类型的对象作为成员变量，并且都包括了完成特定任务的若干回调函数。

！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！







# muduo的回调函数机制

# muduo中的Acceptor and Connector模式

Connector and Acceptor
patterns can help decouple the *connection-related processing*
from the *service processing*, thereby yielding more reusable,
extensible, and efficient communication software.

The Connector
pattern decouples the active establishment of a connection
from the service performed once the connection is estab-
lished.

Acceptor模式的处理过程：
1. 初始化endpoint
2. 初始化服务
3. 服务处理阶段


我们总是在处理客户端向服务器发出请求，服务器再完成响应这种场景。一般的处理方式是服务器

2. 什么是reactor模式
3. reactor模式的组成
4. reactor模式的c++实现
5. muduo的回调函数机制
6. muduo与reactor模式各组件的对应关系（EventLoop/Channel/Acceptor/TcpConnection）
7. 

TcpServer和TcpClient均有TcpConnection的生成及调用，应该分别指的是server_process()和client_process()
TcpConnection: onConnection(), onMessage()


[1] http://cpp.ezbty.org/content/science_doc/libevent%E6%BA%90%E7%A0%81%E6%B7%B1%E5%BA%A6%E5%89%96%E6%9E%90%EF%BC%9Areactor%E6%A8%A1%E5%BC%8F

[2] UNPv3 30章

[3] reactor-simense.pdf

[4] http://daimojingdeyu.iteye.com/blog/828696

[5] http://www.kegel.com/c10k.html

[6] http://blog.csdn.net/solstice/article/details/5848547

[7] Acceptor and Connector
