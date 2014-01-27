#Muduo源码阅读（4）：用于驱动事件循环的EventLoop类

EventLoop类似于Reactor模式中的Initialization Dispatcher，用于与其他模块的调用和交互。EventLoop提供的主要是一个框架，事件的具体处理由Event Handler提供，而事件分发由Demultiplexer提供，但事件的循环、怎样将事件的分发与事件的调用结合起来则是由
EventLoop类决定。打一个比方，EventLoop相当于人脸的轮廓，每个人从远看都差不多，但它决定了人脸大致的结构。Event Handler、Demultiplexer相当于鼻子、眼睛这些细节，正是有了这些细节，才会使人与人之间有种族之分、美丑之分。

##Event Handler的注册与删除
在初始状态，一个EventLoop只是一个框架，其中并没有包含具体需要处理的事件及相应的资源。所以要处理一个事件，必须先将相应的Event Handler信息注册到EventLoop之中，这个用于注册的函数就是void EventLoop::updateChannel()。
```
void EventLoop::updateChannel(Channel* channel) 
{
  assert(channel->ownerLoop() == this);
  assertInLoopThread();
  poller_->updateChannel(channel);
}
```
函数形参为Channel*类型，Channel类似Reactor模式中的Handle，保存了文件描述符、待select的事件等信息，也可认为是事件源。通过Channle类型的变量，Event Handler的相关信息被注册到了EventLoop中。具体的注册过程由poller_成员变量完成，这是Muduo中的Demultiplexer。所以Muduo中Demultiplexer是作为Initialization Dispatcher的成员变量与其交互的。
Event Handler的删除与注册很相似，具体过程也通过poller_完成。
```
void EventLoop::removeChannel(Channel* channel)
{
  ...
  poller_->removeChannel(channel);  //利用poller remove channel
}
```

##事件循环中的事件分发
loop函数是EventLoop类的核心，用于事件循环的运行。在循环结束前，通过Demultiplexer获得已注册到EventLoop中的Event Handler的就绪通知。具体过程请见下文的注释。
```
void EventLoop::loop()
{
  while (!quit_)
  {
    // Demultiplexer将所有就绪的事件Channel保存到activeChannels_中
    pollReturnTime_ = poller_->poll(kPollTimeMs, &activeChannels_); 
 
    // 处理每个就绪的事件，这两行语句证明了EventLoop只提供框架，对要处理的事件一无所知,
    // Channel中包含了感兴趣的事件和对各种事件的处理方法
    for (ChannelList::iterator it = activeChannels_.begin();
        it != activeChannels_.end(); ++it)
    {
      currentActiveChannel_ = *it;   // 处理到来的事件
      currentActiveChannel_->handleEvent(pollReturnTime_);
    }

    //FIXME: 处理事件队列中未来的及处理的事件
    doPendingFunctors();  
  }
}
```

##几个定时运行函数

定时功能由EventLoop中TimerQueue类型的成员变量timerQueue_完成，有以下三个定时运行回调函数的定时函数。
1. runAt()在指定的时间运行回调函数，将延时设置为0。
```
TimerId EventLoop::runAt(const Timestamp& time, const TimerCallback& cb)
{
  return timerQueue_->addTimer(cb, time, 0.0);  //在timerQueue中添加定时器
}
2. runAfter()在当前时间之后延迟若干秒运行回调函数。
TimerId EventLoop::runAfter(double delay, const TimerCallback& cb)
{
  Timestamp time(addTime(Timestamp::now(), delay));
  return runAt(time, cb); 
}
3. runEvery()每隔interval的时间间隔运行回调函数。
TimerId EventLoop::runEvery(double interval, const TimerCallback& cb)
{
  Timestamp time(addTime(Timestamp::now(), interval));
  return timerQueue_->addTimer(cb, time, interval);    //添加一个以一定间隔运行的定时器
}
```

##在事件循环中运行回调函数
EventLoop::runInLoop()用于在一个事件循环中运行回调函数。如果事件循环在当前线程中，那么可以直接运行回调函数。但如果不在当前线程，那么将回调函数放入待完成事件集合中，在loop()函数中的doPendingFunctors()完成处理。在上文中可以看到loop函数的每次while()循环在最后都会运行doPendingFunctors()。
```
void EventLoop::runInLoop(const Functor& cb)
{
  if (isInLoopThread()) //事件循环线程是当前线程
  {
    cb();   //调用函数
  }
  else
  {
    // 将回调函数放入EventLoop的待完成事件中
    queueInLoop(cb);  
  }
}
```
queueInLoop()主要完成的任务是将回调函数放入pendingFunctors_暂存。
```
void EventLoop::queueInLoop(const Functor& cb)
{
  ...
  pendingFunctors_.push_back(cb);  
  ...

}
```


pendingFunctors/_保存了待完成的事件，doPendingFunctors()函数很简单，就是顺序调用pendingFunctors/_各个回调函数。
```
void EventLoop::doPendingFunctors()
{
  ...
  functors.swap(pendingFunctors_);
  for (size_t i = 0; i < functors.size(); ++i)
  {
    functors[i]();//逐个调用Pending Functors
  }
  ...
}
```

？？？？？？？？？？？？？
EventLoop::wakeup()；




