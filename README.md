# mymuduo
## muduo库核心机制分析
### Reactor模型的实现
#### 1.Reactor模型基础
重要组件：Event事件、Reactor反应堆、Demultiplex事件分发器、Evanthandler事件处理器

模型图：

#### 2.muduo中Reactor模型的实现
在muduo中，EventLoop作为反应堆，EpollPoller作为事件分发器（基于epoll封装），Channel作为事件处理器。

Reactor模型执行流程：

- 创建EventLoop对象loop，并调用loop.loop()

   - 初始化loop时，将会调用EventLoop的构造函数，EventLoop的构造函数将初始化EpollPoller对象poller_ ====>  将调用epoll_create
   - loop.loop() ===> 将调用poller_->poll() ===> 将调用epll_wait

- 创建Channel对象，并在通过EventLoop在poller_中注册事件，并设置回调函数

  - Channel在初始化时，需要传入EventLoop对象  Channel::Channel(EventLoop *loop, int fd)
  - Channel->enableReading()/disableReading()注册或删除读事件   ===> loop_->updateChannel() ===> poller_->update() ===> 将调用epoll_ctl
  - Channel->setReadCallback() 设置读事件发生后的回调函数
 
- 事件发生时，执行回调
  - loop.loop()函数中的核心部分
  
  ```
        pollReturnTime_ = poller_->poll(kPollTimeMs, &activeChannels_);
        
        for (Channel *channel : activeChannels_)
        {
            channel->handleEvent(pollReturnTime_);
        }
   ```
   - handEvent将根据具体发生的事件去调用注册的回调函数

#### 3.核心问题：事件发生时返回的epoll_event，如何转化为channel?
  首先在poller_->update时会做一个特殊处理让event.data.ptr指向channel对象
  ```
  void EPollPoller::update(int operation, Channel *channel)
{
    epoll_event event;
    bzero(&event, sizeof event);
    
    int fd = channel->fd();

    event.events = channel->events();
    event.data.fd = fd; 
    event.data.ptr = channel;//关键步骤
    
    if (::epoll_ctl(epollfd_, operation, fd, &event) < 0)
    {
        if (operation == EPOLL_CTL_DEL)
        {
            LOG_ERROR("epoll_ctl del error:%d\n", errno);
        }
        else
        {
            LOG_FATAL("epoll_ctl add/mod error:%d\n", errno);
        }
    }
}
  
  ```
  在poller->poll()中，将调用fillActiveChannel函数将找到channel对象的指针
  ```
  Timestamp EPollPoller::poll(int timeoutMs, ChannelList *activeChannels)
{
    // 实际上应该用LOG_DEBUG输出日志更为合理
    LOG_INFO("func=%s => fd total count:%lu \n", __FUNCTION__, channels_.size());

    int numEvents = ::epoll_wait(epollfd_, &*events_.begin(), static_cast<int>(events_.size()), timeoutMs);//关键步骤
    int saveErrno = errno;
    Timestamp now(Timestamp::now());

    if (numEvents > 0)
    {
        LOG_INFO("%d events happened \n", numEvents);
        fillActiveChannels(numEvents, activeChannels);//关键步骤
        if (numEvents == events_.size())
        {
            events_.resize(events_.size() * 2);
        }
    }
 ...略
}
  ```
  
  
  ```
  void EPollPoller::fillActiveChannels(int numEvents, ChannelList *activeChannels) const
{
    for (int i=0; i < numEvents; ++i)
    {
        Channel *channel = static_cast<Channel*>(events_[i].data.ptr);
        channel->set_revents(events_[i].events);
        activeChannels->push_back(channel); // EventLoop就拿到了它的poller给它返回的所有发生事件的channel列表了
    }
}
 ```
