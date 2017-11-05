---
layout: post
title: Reactor模式解析——muduo网络库
category: notes
tags: 网络编程 linux socket C++
---

* 文章目录
{:toc}

最近一段时间阅读了muduo源码，读完的感受有一个感受就是有点乱。当然不是说代码乱，是我可能还没有完全消化和理解。为了更好的学习这个库，还是要来写一些东西促进一下。

我一边读一边尝试在一些地方改用c++11的新特性，这个工作持续在进行中。为啥这么干？没什么理由，纯粹是为了学习。

注：本文的大部分代码和图文都来自《Linux多线程服务端编程》，可直接参考[muduo][1]的源码,或者参考我[这里][2]抄着玩儿的版本。

## Reactor介绍
什么是Reactor？ 换个名词“non-blocking IO + IO multiplexing”，意思就显而易见了。Reactor模式用非阻塞IO+poll(epoll)函数来处理并发，程序的基本结构是一个事件循环，以事件驱动和事件回调的方式实现业务逻辑。
{% highlight c++ linenos %}
  while(!done)
  {
      int retval  = poll(fds,nfds,timeout)
      if(retval < 0)
          处理错误，回调用户的error handler
      else{
          处理到期的timers,回调用户的timer handler
          if(retval > 0){
              处理IO事件，毁掉用户的IO event handler
          }
      }
  }
{% endhighlight %}

这段代码形式上非常简单，跟我上一篇文章epoll的例子十分相似，除了没有处理超时timer部分。在muduo的实现中，定时器使用了linux平台的timerfd_*系列函数, timers和其它IO统一了起来。

## 单线程Reactor实现
muduo的Reactor核心主要由Channel、EventLoop、Poller、TimerQueue这几个类完成。乍一看还有一点绕，代码里面各种回掉函数看起来有点不直观。另外，这几个类的生命周期也值得注意，容易理不清楚。
### 1. Channel
Channel类比较简单，负责IO事件分发，每一个Channel对象都对应了一个fd，它的核心成员如下：
{% highlight c++ linenos %}
  EventLoop* loop_;
  const int fd_;
  int events_;
  int revents_;
  int index_;

  ReadEventCallback readCallback_;
  EventCallback writeCallback_;
  EventCallback errorCallback_;
  EventCallback closeCallback_;
{% endhighlight %}
几个callback函数都是c++新标准里面的function对象（muduo里面是Boost::function），它们会在handleEvent这个成员函数中根据不同的事件被调用。index_是poller类中pollfds_数组的下标。events_和revents_明显对应了struct pollfd结构中的成员。需要注意的是，Channel并不拥有该fd,它不会在析构函数中去关闭这个fd（fd是由Socket类的析构函数中关闭，即RAII的方法），Channel的生命周期由其owner负责。

### 2. Poller
Poller类在这里是poll函数的封装（在muduo源码里面是抽象基类，支持poll和epoll），它有两个核心的数据成员：
{% highlight c++ linenos %}
  typedef std::vector<struct pollfd> PollFdList;
  typedef std::map<int, Channel*> ChannelMap;  // fd to Channel
  PollFdList pollfds_;
  ChannelMap channels_;
{% endhighlight %}
ChannelMap是fd到Channel类的映射，PollFdList保存了每一个fd所关心的事件，用作参数传递到poll函数中，Channel类里面的index_即是这里的下标。Poller类有下面四个函数
{% highlight c++ linenos %}
  Timestamp poll(int timeoutMs, ChannelList* activeChannels);
  void updateChannel(Channel* channel);
  void removeChannel(Channel* channel);
  private:
  void fillActiveChannels(int numEvents, ChannelList* activeChannels) const;
{% endhighlight %}
updateChannel和removeChannel都是对上面两个数据结构的操作，poll函数是对::poll的封装。私有的fillActiveChannels函数负责把返回的活动时间添加到activeChannels（vector<Channel*>）这个结构中，返回给用户。Poller的职责也很简单，负责IO multiplexing，一个EventLoop有一个Poller，Poller的生命周期和EventLoop一样长。

### 3. EventLoop
EventLoop类是核心，大多数类都会包含一个EventLoop*的成员，因为所有的事件都会在EventLoop::loop()中通过Channel分发。先来看一下这个loop循环:
{% highlight c++ linenos %}
  while (!quit_)
  {
      activeChannels_.clear();
      pollReturnTime_ = poller_->poll(kPollTimeMs, &activeChannels_);
      for (ChannelList::iterator it = activeChannels_.begin();
          it != activeChannels_.end(); ++it)
      {
      (*it)->handleEvent(pollReturnTime_);
      }
      doPendingFunctors();
  }
{% endhighlight %}
handleEvent是Channel类的成员函数，它会根据事件的类型去调用不同的Callback。循环末尾还有一个doPendingFunctors()，这个函数的作用在后面多线程的部分去说明。
由上面三个类已经可以构成Reactor的核心，整个流程如下：
- 用户通过Channel向Poller类注册fd和关心的事件
- EventLoop从poll中返回活跃的fd和对应的Channel
- 通过Channel去回掉相应的时间。

muduo的书里面有一个时序图（8-1），很清楚的说明了整个流程。
### 4. TimerQueue
muduo的定时器直接使用了标准的容器库set来管理。先看一下TimerQueue:
{% highlight c++ linenos %}
  typedef std::shared_ptr<Timer> TimerPtr;
  typedef std::pair<Timestamp, TimerPtr> Entry;
  typedef std::set<Entry> TimerList;
  Channel timerfdChannel_;
  const int timerfd_;
  TimerList timers_;
{% endhighlight %}
采用std::pair<Timestamp, TimerPtr> 加上set的 的形式是为了处理两个Timer同事到期的情况,即使到期时间相同,它们的地址也不同。timerfdChannel_是用来管理timerfd_create函数创建的fd。Timer类里面包含了一个回调函数和一个到期时间。expiration_就是上面Entry中的Timestamp。
{% highlight c++ linenos %}
  const TimerCallback callback_;
  Timestamp expiration_;
{% endhighlight %}
这样整个思路就很清晰了：
- 用一个set来保存所有的事件和时间
- 根据set集合里面最早的时间来更新timerfd_的到期时间（用timerfd_settime函数）
- 时间到期后，EventLoop的poll函数会返回，并调用timerfdChannel_里面的handleEvent回调函数。
- 通过handleEvent这个回调函数，再去处理到期的所有事件。
{% highlight c++ linenos %}
  timerfdChannel_.setReadCallback(
      std::bind(&TimerQueue::handleRead,this));
  timerfdChannel_.enableReading();
{% endhighlight %}
timerfdChannel_的callback函数注册了TimerQueue的handleRead函数。在handleRead中应该干什么就很明显了，自然是捞出所有到期的timer，一次去执行对应的事件：
{% highlight c++ linenos %}
  void TimerQueue::handleRead()
  {
      loop_->assertInLoopThread();
      Timestamp now(Timestamp::now());
      readTimerfd(timerfd_, now);
      std::vector<Entry> expired = getExpired(now);
      // safe to callback outside critical section
      for (std::vector<Entry>::iterator it = expired.begin();
          it != expired.end(); ++it)
      {
          it->second->run();
      }
      reset(expired, now);
  }
{% endhighlight %}
至此为止，单线程的Reator就已经完成了。总感觉muduo这种事件回调的代码风格，读起来比较绕，不够直观。不知道其他的Reactor模式的网络程序会不会也是这种感觉。

## 多线程的技巧

多线程本质上是困难的，因为它强迫你的大脑去思考两件事情同时发生会出现的各种情况。我目前感觉除了看别人的经验和总结，没有什么技巧或者方法论来解决多线程的问题。

一个线程一个EventLoop，每个线程都有自己管理的各种ChannelList和TimerQueue。有时候，我们总有一些需求，要在各个线程之间调配任务。比如添加一个定时时间到IO线程中，这样TimerQueue就有两个线程同时访问。我认为muduo在处理上锁的问题上，很值得学习。
### 1. runInLoop()和queueInLoop()
先来看几个EventLoop里面重要的函数和成员：
{% highlight c++ linenos %}
  std::vector<Functor> pendingFunctors_; // @GuardedBy mutex_
  void EventLoop::runInLoop(Functor&& cb) {
    if (isInLoopThread()) {
      cb();
    } else {
      queueInLoop(std::move(cb));
    }
  }
  void EventLoop::queueInLoop(Functor&& cb) {
    {
      MutexLockGuard lock(mutex_);
      pendingFunctors_.push_back(cb);
    }
    if (!isInLoopThread() || callingPendingFunctors_) {
      wakeup();
    }
  }
{% endhighlight  %}
注意这里的函数参数，我用到了C++11的右值引用。
在前面的EventLoop::loop里面，我们已经看到了doPendingFunctors()这个函数，EventLoop还有一个重要的成员pendingFunctors_，该成员是暴露给其他线程的。这样，其他线程向IO线程添加定时时间的流程就是：
- 其他线程调用runInLoop(),
- 如果不是当前IO线程，再调用queueInLoop()
- 在queueLoop中，将时间push到pendingFunctors_中，并唤醒当前IO线程
注意这里的唤醒条件：不是当前IO线程肯定要唤醒；此外，如果正在调用Pending functor，也要唤醒；（为什么？，因为如果正在执行PendingFunctor里面，如果也执行了queueLoop，如果不唤醒的话，新加的cb就不会立即执行了。）

### 2.doPendingFunctors()
现在来看一下doPendingFunctors()这个函数：
{% highlight c++ linenos %}
  void EventLoop::doPendingFunctors() {
    std::vector<Functor> functors;
    callingPendingFunctors_ = true;
    {
      // reduce the lenth of the critical section
      // avoid the dead lock cause the functor can call queueInloop(;)
      MutexLockGuard lock(mutex_);
      functors.swap(pendingFunctors_);
    }
    for (size_t i = 0; i < functors.size(); ++i) {
      functors[i]();
    }
    callingPendingFunctors_ = false;
  }
{% endhighlight %}
doPendingFunctors并没有直接在临界区去执行functors,而是利用了一个栈对象，把事件swap到栈对象中，再去执行。这样做有两个好处：
1. 减少了临界区的长度，其它线程调用queueInLoop对pendingFunctors加锁时，就不会被阻塞
2. 避免了死锁，可能在functors里面也会调用queueInLoop()，从而造成死锁。

回过头来看，muduo在处理多线程加锁访问共享数据的策略上，有一个很重要的原则:**拼命减少临界区的长度** 
试想一下，如果没有pendingFunctors_这个数据成员，我们要想往TimerQueue中添加timer，肯定要对TimerQueue里面的insert函数加锁，造成锁的争用，而pendingFunctors_这个成员将锁的范围减少到了一个vector的push_back操作上。此外，在doPendingFunctors中，利用一个栈对象减少临界区，也是很巧妙的一个重要技巧。

### 3.wake()
前面说到唤醒IO线程，EventLoop阻塞在poll函数上，怎么去唤醒及时它？以前的做法是利用pipe,向pipe中写一个字节，监视在这个pipe的读事件的poll函数就会立刻返回。在muduo中，采用了linux中eventfd调用
{% highlight c++ linenos %}
  static int createEventfd() {
    int evtfd = ::eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
    if (evtfd < 0) {
      LOG_SYSERR << "Failed in eventfd";
      abort();
    }
    return evtfd;
  }
  void EventLoop::wakeup() {
    uint64_t one = 1;
    ssize_t n = ::write(wakeupFd_, &one, sizeof one);
    if (n != sizeof one) {
      LOG_ERROR << "EventLoop::wakeup() writes " << n << " bytes instead of 8";
    }
  }
{% endhighlight %}
把eventfd得到的fd和前面一样，通过Channel注册到poll里面，唤醒的时候，只需要向wakeupFd中写入一个字节，就能达到唤醒的目的。eventfd、timerfd都体现了linux的设计哲学，Everyting is a fd。

关于muduod的Reactor模式，现在我终于有一些理解了。关于TCPServer部分，下一篇再写。下一步，我会继续研究muduo的HTTP示例，并尝试扩展它。


  [1]: https://github.com/chenshuo/muduo
  [2]: https://github.com/xibaohe/muduo_clone