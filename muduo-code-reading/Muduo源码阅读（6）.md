#Muduo源码阅读（6）：事件多路分发机制
Muduo中的事件多路分发主要是通过poll完成，也是Muduo中少有的使用到继承的模块。在这一部分中，Poller类为接口，PollPoller和EPollPoller为具体的实现，分别是linux中poll、epoll的封装。
#1 Poller接口
Poller接口并不复杂，主要包括三个成员函数:
* *poll()* —— 获取当前就绪的Channel。
* *updateChannel()* —— Channel的注册。这里的*updateChannel()*完成实际的注册工作，前面在EventLoop、Channel中类似的函数只是调用而已。具体的调用层次为：*Channel::update() -> EventLoop::updateChannel() -> Poller::updateChannel()*。
* *removeChannel()* —— Channel的删除。调用层次类似*updateChannel()*。
```
class Poller : boost::noncopyable
{
 public:
  typedef std::vector<Channel*> ChannelList;

  /// Polls the I/O events.
  /// Must be called in the loop thread.
  virtual Timestamp poll(int timeoutMs, ChannelList* activeChannels) = 0;

  /// Changes the interested I/O events.
  /// Must be called in the loop thread.
  virtual void updateChannel(Channel* channel) = 0;

  /// Remove the channel, when it destructs.
  /// Must be called in the loop thread.
  virtual void removeChannel(Channel* channel) = 0;

  static Poller* newDefaultPoller(EventLoop* loop);

 private:
  EventLoop* ownerLoop_;
};
```

#2 PollPoller类
PollPoller是封装linux中poll函数的类，主要的功能有二：一是poll函数的调用，二是Channel在Poller中的注册。在这两部分功能中，为了更新相应Channel和文件描述符的状态并建立二者的对应关系使用到了pollfds_、channels_成员变量。
##2.1 PollPoller::poll()函数
这个函数先是通过linux的poll函数测试pollfds_中保存的文件描述符及相应的事件，再调用*fillActiveChannels()*将就绪的Channel保存到activeChannels中。
```
Timestamp PollPoller::poll(int timeoutMs, ChannelList* activeChannels)
{
  int numEvents = ::poll(&*pollfds_.begin(), pollfds_.size(), timeoutMs);
  .... 
  fillActiveChannels(numEvents, activeChannels);
  ...
}
```
*fillActiveChannels()*遍历所有已注册事件的struct pollfd，判断是否有事件在其上发生。如果有，那么通过文件描述符找到对应的Channel，根据struct pollfd更新其到来事件的状态并放入activeChannels中。这里Channel通过map<int,Channel*>类型的数据结构，既保存Channel信息也建立了文件描述符与Channel之间的关系。
```
void PollPoller::fillActiveChannels(int numEvents,
                                    ChannelList* activeChannels) const
{
  for (PollFdList::const_iterator pfd = pollfds_.begin();
      pfd != pollfds_.end() && numEvents > 0; ++pfd)
  {
    // 若有事件发生
    if (pfd->revents > 0)
    {
      --numEvents;
      ChannelMap::const_iterator ch = channels_.find(pfd->fd);
      assert(ch != channels_.end());
      Channel* channel = ch->second;   //通过文件描述符找到对应的channel对象
      channel->set_revents(pfd->revents); //设置channel对象的状态，令文件描述符和其对应的channel状态对应上
      activeChannels->push_back(channel);
    }
  }
}
```
##2.2 PollPoller::updateChannel()函数
*updateChannel()*将Channel注册到Poller中，需要改变Poller中保存Channel相关信息的两个成员：pollfds_、channels_。如果当前Channel未注册到Poller中那么直接添加，如果Channel已经注册到了Poller中，那么用其当前信息更新Poller内部保存的信息。
```
void PollPoller::updateChannel(Channel* channel)
{
  if (channel->index() < 0)   //Channel的默认构造函数index_为-1
  {
    // a new one, add to pollfds_
    struct pollfd pfd;
    pfd.fd = channel->fd();
    pfd.events = static_cast<short>(channel->events());
    pfd.revents = 0;
    pollfds_.push_back(pfd);
    int idx = static_cast<int>(pollfds_.size())-1;
    channel->set_index(idx);//channel的index就是与其对应的pollfd在数组中的位置
    channels_[pfd.fd] = channel; //另外用map建立channel与fd的对应关系
    //通过这两个数据结构，可以通过fd找到对应的channel，也可以通过channel找到对应的pollfd结构
  }
  else
  {
    // update existing one
    //根据channel更新PollPoller中pollfd结构的状态
    assert(channels_.find(channel->fd()) != channels_.end());
    assert(channels_[channel->fd()] == channel);
    int idx = channel->index();
    assert(0 <= idx && idx < static_cast<int>(pollfds_.size()));
    struct pollfd& pfd = pollfds_[idx];
    assert(pfd.fd == channel->fd() || pfd.fd == -channel->fd()-1);
    pfd.events = static_cast<short>(channel->events());
    pfd.revents = 0;
    if (channel->isNoneEvent())
    {
      // ignore this pollfd
      pfd.fd = -channel->fd()-1;
    }
  }
}
```


