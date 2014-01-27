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
*updateChannel()*将Channel注册到Poller中，需要改变Poller中保存Channel相关信息的两个成员：pollfds_、channels_。如果当前Channel未注册到Poller中那么直接添加即可，如果Channel已经注册到了Poller中，那么用Channel当前信息更新Poller内部保存的信息。
```
void PollPoller::updateChannel(Channel* channel)
{
  //Channel的默认构造函数index_为-1，所以下面的语句判断当前channel是否已经添加到Poller中
  if (channel->index() < 0)   
  {
    // 根据Channel填充struct pollfd信息，并加入pollfds_
    struct pollfd pfd;
    pfd.fd = channel->fd();
    pfd.events = static_cast<short>(channel->events());
    pfd.revents = 0;
    pollfds_.push_back(pfd);

    // idx为channel在Poller中pollfds_的索引，用于更改Channel的index_变量
    int idx = static_cast<int>(pollfds_.size())-1;
    channel->set_index(idx);
    // 将当前Channel插入channels_，对应的所引为pfd.fd
    channels_[pfd.fd] = channel; 
    //有了这两个数据结构，既可以通过fd找到对应的channel，也可以通过channel找到对应的pollfd结构
  }
  else
  {
    // 若当前Channel已经在Poller中存在，则更新Poller中pollfd结构的状态
    // 获得Channel在pollfds_中的索引idx，首次插入时分配
    int idx = channel->index();
    struct pollfd& pfd = pollfds_[idx];
    pfd.events = static_cast<short>(channel->events());
    pfd.revents = 0;
  }
}
```

##2.3 PollPoller::removeChannel()函数
类似Channel的注册，Channel的删除同样需要更改pollfds_、channels_这两个成员。
```
void PollPoller::removeChannel(Channel* channel)
{
  // 获得Channel在pollfds_中的索引
  int idx = channel->index();
  const struct pollfd& pfd = pollfds_[idx]; (void)pfd;
  // 先从channels_中删除Channel
  size_t n = channels_.erase(channel->fd());
  // 如果Channel对应的pollfd结构在pollfds_数组的末尾，那么直接删除即可
  if (implicit_cast<size_t>(idx) == pollfds_.size()-1)
  {
    pollfds_.pop_back();
  }
  else
  {
    // 如果要删除的Channel其对应的pollfd结构不在pollfds_的末尾，
    // 那么需要将对应的pollfd结构与末尾元素交换，修改原末尾元素对应Channel
    // 的pollfd索引位置信息，再通过pop_back()删除当前Channel对应的pollfd结构
    int channelAtEnd = pollfds_.back().fd;
    iter_swap(pollfds_.begin()+idx, pollfds_.end()-1);
    if (channelAtEnd < 0)
    {
      channelAtEnd = -channelAtEnd-1;
    }
    channels_[channelAtEnd]->set_index(idx);
    pollfds_.pop_back();
  }
}
```


