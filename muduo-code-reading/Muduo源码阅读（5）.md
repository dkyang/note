#Muduo源码阅读（5）：Muduo中的Handle —— Channel类
Channel在Muduo中起到类似Reactor模式中Handle的作用，但与Hanndle仅仅标识系统资源不同，Channel还定义了感兴趣的事件和对各种类型事件的处理方式。所谓系统资源，在Windows系统里是句柄和套接字，在Linux系统里指文件描述符，Channel类中就包含了fd_成员变量用于保存相应I/O过程的文件描述符。

##1 作为中介与抽象的Channel
Channel在Event Handler与Dispatcher之间起到中介的作用。一方面可与Event Handler交互，Event Handler设置Channel的以下几方面内容:
1. 与I/O相关的文件描述符，相应成员变量为fd_。
2. 在当前Channel上感兴趣的事件，相应成员变量为events_，在Event Handler中通过调用Channel的enableReading()、enableWriting()函数设置。
3. 处理各种类型事件的回调函数,相应成员变量包括readCallback/_，writeCallback/_，closeCallback/_，errorCallback_，在Event Handler中通过调用Channel的接口函数setReadCallback()等完成。这里与传统Reactor模式实现不同的是，Event Handler不仅将事件相关的`数据`传给了Handle，也设置了Handle对各种事件的`操作`。这样有了Channel后，Dispatcher完全无需知道Event Handler的存在，只需与相应的Channel交互即可。这种实现之所以可行，也是由Muduo采用的boost::function+boost::bind回调函数机制决定，事件处理的函数成为了变量可以任意的传递。

另一方面，Dispatcher(即EventLoop)与Channel通信，包括以下几个方面的内容：
1. Channel在EventLoop中的注册。
```
void Channel::update()
{
  loop_->updateChannel(this);
}
```
Channel包含EventLoop类型的成员变量loop_，在update()函数中调用loop的Channel注册函数updateChannel()将自身注册到EventLoop中。
2. 通知在当前Channel中就绪的事件。EventLoop通过Demultiplexer得到就绪Channel中发生的事件，并将其保存在Channel的revents_成员变量中。
3. Channel事件处理函数handleEvent()的调用。EventLoop会调用就绪Channel的事件处理函数，Channel根据revents_的值调用Event Handler传入的相应回调函数。

可见Channel是Event Handler与Dispatcher之间的桥梁，二者不直接通信，而是分别单方与Channel交互。其中的成员变量fd/_、events/_、revents/_正好对应Linux中struct pollfd的三个成员，实际上也起到了类似的作用。

##2 相关成员函数
在上文中总体介绍Channel的各部分功能，现在让我们分成员函数具体介绍比较重点的几个。
###2.1 handleEvent
*handleEvent()*在EventLoop的事件循环*loop()*中被调用，也是Channle比较重要的一个功能，具体的事件处理通过*handleEventWithGuard()*完成。
```
void Channel::handleEvent(Timestamp receiveTime)
{
    ...  
    handleEventWithGuard(receiveTime);
    ...
}
```
函数中的POLLIN、POLLNVAL、POLLERR都是poll函数的测试条件，*handleEventWithGuard()*根据这些测试条件调用相应的回调函数。比如在发生数据可写事件时调用写回调函数、在发生错误或描述符不是一个打开文件时调用错误处理回调函数。
```
void Channel::handleEventWithGuard(Timestamp receiveTime)
{
  if ((revents_ & POLLHUP) && !(revents_ & POLLIN))   //POLLUP发生挂起
  {
    if (closeCallback_) closeCallback_(); 
  }

  if (revents_ & POLLNVAL) 
  {
    LOG_WARN << "Channel::handle_event() POLLNVAL";
  }

  if (revents_ & (POLLERR | POLLNVAL)) //若发生错误或描述符不是一个打开文件
  {
    if (errorCallback_) errorCallback_();
  }
  if (revents_ & (POLLIN | POLLPRI | POLLRDHUP))
  {
    if (readCallback_) readCallback_(receiveTime);
  }

  if (revents_ & POLLOUT) //若普通数据可写
  {
    if (writeCallback_) writeCallback_();
  }
}
```

###2.2 设置回调函数的接口
这几个函数在功能上比较重要，是Event Handler设置各种事件处理回调函数的接口。但形式上很简单，仅仅是参数的传递。
```
  // 读回调函数设置 
  void setReadCallback(const ReadEventCallback& cb)
  { readCallback_ = cb; }
  // 写回调函数设置 
  void setWriteCallback(const EventCallback& cb)
  { writeCallback_ = cb; }
  // Channel关闭回调函数设置 
  void setCloseCallback(const EventCallback& cb)
  { closeCallback_ = cb; }
  // 错误处理回调函数设置 
  void setErrorCallback(const EventCallback& cb)
  { errorCallback_ = cb; }
```

###2.3 感兴趣事件设置 
下面的四个函数同属一个类型，用于设置当前Channel是否关心读、写事件的发生，另外通过update()函数或者将Channel插入Demultiplexer，或者更新Demultiplexer中相应Channel的状态。
```
  const int Channel::kNoneEvent = 0;   
  const int Channel::kReadEvent = POLLIN | POLLPRI;
  const int Channel::kWriteEvent = POLLOUT; 

  void enableReading() { events_ |= kReadEvent; update(); }
  void enableWriting() { events_ |= kWriteEvent; update(); }
  void disableWriting() { events_ &= ~kWriteEvent; update(); }
  void disableAll() { events_ = kNoneEvent; update(); }
```




 
