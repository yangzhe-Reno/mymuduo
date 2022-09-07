# mymuduo
## muduo库核心机制分析
### (1)Reactor模型的实现
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
 
#### 4.总结
muduo库中提供的reactor模型基本API总结如下：
```
//创建反应堆
Eventloop loop;

//创建事件执行器
int fd = socket()
Channel channel(loop, fd);
channel->enableReading() //注册读事件
Channel->setReadCallback(f) //设置读事件回调函数f
//反应堆运行
loop.loop()

```
### (2)主从Reactor模型的实现
需要考虑的几个问题：
- 如何管理一组相同的线程？  
- epoll_wait()系统调用将会导致线程阻塞，如何将其唤醒？

第一个问题，muduo库设计了线程池来对一组子线程管理，该子线程都将运行上述所说的loop.loop()函数，详细设计请自行看EventLoopThreadPool 等几个类。
在EventLoopThreadPool中，muduo库提供了一个函数getNextLoop()，可以通过轮询算法依次获取到线程池中各线程的EvenLoop对象。
```
EventLoop* EventLoopThreadPool::getNextLoop()
{
    EventLoop *loop = baseLoop_;

    if (!loops_.empty()) // 通过轮询获取下一个处理事件的loop
    {
        loop = loops_[next_];
        ++next_;
        if (next_ >= loops_.size())
        {
            next_ = 0;
        }
    }

    return loop;
}
```
对于第二个问题如何唤醒子线程，分析步骤如下：

- 在EventLoop类中设计了两个成员变量
```
    int wakeupFd_;
    std::unique_ptr<Channel> wakeupChannel_;
```
其中wakeupFd_是特殊的eventfd，具体可见代码：
```
int createEventfd()
{
    int evtfd = ::eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
    if (evtfd < 0)
    {
        LOG_FATAL("eventfd error:%d \n", errno);
    }
    return evtfd;
}

EventLoop::EventLoop()
...
    , poller_(Poller::newDefaultPoller(this))
    , wakeupFd_(createEventfd())
    , wakeupChannel_(new Channel(this, wakeupFd_))
{
...
    wakeupChannel_->setReadCallback(std::bind(&EventLoop::handleRead, this));
    wakeupChannel_->enableReading();
}
```
通过上面的代码，在调用EventLoop的构造函数时，创建了wakeupFd_和wakeupChannel_，并对wakeupChannel_设置了读事件回调函数和注册读事件。

所以当wakeupFd_读事件发生时，即有数据写入到wakeupFd_，将会调用EventLoop::handleRead。
```
void EventLoop::handleRead()
{
  uint64_t one = 1;
  ssize_t n = read(wakeupFd_, &one, sizeof one);
  if (n != sizeof one)
  {
    LOG_ERROR("EventLoop::handleRead() reads %lu bytes instead of 8", n);
  }
}
```
由上可见EventLoop::handleRead()并没有做什么实际的事情，因为可以理解为当有人向wakeupFd_写入数据时，原本因为loop.loop()阻塞（运行到epoll_wait时，无时间发生就会阻塞）的线程将被epoll通知wakeupFd_上有数据写入，发生了读事件，需要调用EventLoop::handleRead()处理，于是线程解除了阻塞。

什么时候wakeupFd_将会有数据写入呢？通过queueInLoop调用到wakeup()
```
void EventLoop::queueInLoop(Functor cb)
{
    {
        std::unique_lock<std::mutex> lock(mutex_);
        pendingFunctors_.emplace_back(cb);
    }


    if (!isInLoopThread() || callingPendingFunctors_) 
    {
        wakeup(); // 唤醒loop所在线程
    }
}

void EventLoop::wakeup()
{
    uint64_t one = 1;
    ssize_t n = write(wakeupFd_, &one, sizeof one);
    if (n != sizeof one)
    {
        LOG_ERROR("EventLoop::wakeup() writes %lu bytes instead of 8 \n", n);
    }
}
```
所以连起来看，当主线程如何唤醒子线程？即在主线程中，通过getNextLoop()函数获得子线程的loop对象，再在主线程中loop->queueInLoop(f)，如果此时子线程被阻塞将会触发wakeuo()，子线程被唤醒后，将会执行主线程传递的函数f。
