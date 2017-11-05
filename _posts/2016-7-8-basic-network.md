---
layout: post
title: 基本网络编程范式
category: notes
tags: 网络编程 linux
excerpt: 本文是自己学习经验总结，有不正确的地方，请批评指正。
---

* 目录
{:toc}

总结一下这一段时间来，有关网络编程的学习。我是从csapp的最后章节的Tiny HTTP服务器开始，以它为基础，改用不同的方式实现并发，包括进程、线程、线程池、I/O多路复用。所有代码见地址：https://github.com/xibaohe/tiny_server

## 一、基于进程、线程的并发
关于进程和线程的网络编程模型，在UNP卷1的第30章，有详细的介绍。我这里，在Tiny基础上，实现了以下几种：

- tiny_process：每个连接开一个进程
- tiny_thread：每个连接开一个线程
- tiny_thread_pre：事先创建线程池，由主线程同一accept，fdbuffer采用信号量同步(同csapp第12章)
- tiny_thread_mutex：同上，fdbuffer采用互斥锁和条件变量实现
- tiny_thread_pre2：事先创建线程池，每个线程各自accept。
    
其中，fdbuffer是指主线程accept得到已连接描述符后，存放进fdbuffer缓冲区，其他线程再去处理。
    
### 多进程

{% highlight c++ linenos %}
    Signal(SIGPIPE,SIG_IGN);//忽略SIGPIPE,见UNP 5.13
    Signal(SIGCHLD,sigchld_hander);//回收子进程
    listenfd = Open_listenfd(port);//见csapp相关章节
    while (1) {
        connfd = Accept(listenfd, (SA *)&clientaddr,&clientlen);
        if(Fork() == 0){//the children process
            Close(listenfd);//子进程关闭监听描述符
            doit(connfd);//子进程处理fd
            Close(connfd);
            exit(0);
        }
        Close(connfd);               
    }
    void sigchld_hander(int sig)
    {
        while(waitpid(-1,0,WNOHANG) > 0)
        {
            if(verbose)
            printf("a child process gone!!\n");
        }
        return;
    }
{% endhighlight %}

上述代码是最简单的并发模型，每一个连接，都会新fork一个进程去处理，显然这种方式并发程度低。对于初学者，还是有几个需要注意的地方。

1. 信号处理，SIGPIPE（向已经关闭的fd写数据）默认会终止进程，这里忽略它。sigchld_hander用于函数回收子进程(注意信号不排队问题)。
2. 注意父子进程都需要关闭已连接描述符connfd,到客户端的连接才会最终关闭

注：doit函数来自于csapp的Tiny服务器，我添加了对HEAD、POST方法的简单支持，详细请参考全部源码。

### 多线程
{% highlight c++ linenos %}
    while (1) {
        connfd = Malloc(sizeof(int));//avoid race condition
        *connfd = Accept(listenfd, (SA *)&clientaddr,&clientlen);
        Pthread_create(&tid,NULL,&thread,connfd);             
    }
    void *thread(void *vargp)
    {
        int connfd = *((int *)vargp);
        Pthread_detach(pthread_self());
        Free(vargp);
        doit(connfd);
        Close(connfd);
        return NULL;
    }
{% endhighlight %}

多线程与多进程基本一致，需要注意的地方：
1. 向线程传递connfd的race condition：如果我们不申请内存，直接传递&connfd给进程，在线程从vargp获取connfd时， connfd的值可能被主线程新accept的值替换了。
2. 线程是可结合或者是分离的，区别在于分离式线程的存储器资源在它自己终止的时候由系统自动释放，而可结合线程需要其他线程回收，此处的Pthread_detach是将当前线程分离。

### 预先创建线程
为每一个客户都创建一个新的线程，显然不是高效的做法，我们可以预先创建线程，主线程和其它线程通过一个缓冲区传递描述符，或者可以每个线程自己accept。

##### 信号量同步
{% highlight c++ linenos %}
    int i;
    for(i=0;i<NTHREADS;i++)/*create worker threads*/
        Pthread_create(&tid,NULL,thread,NULL);
    while (1) {
        connfd = Accept(listenfd, (SA *)&clientaddr,&clientlen);
        sbuf_insert(&sbuf,connfd);     
    }
    void *thread(void *vargp)
    {
        Pthread_detach(pthread_self());
        while(1)
        {
          int connfd = sbuf_remove(&sbuf);
          doit(connfd);
          Close(connfd);
        }
    }
{% endhighlight %}
首先创建固定数量的线程，主线程将已连接描述符放如缓冲区中，其它线程再从缓冲区中取出fd,并处理。这是一个典型的生产者和消费者问题，在这个版本中，采用csapp中的信号量来解决同步问题，缓冲区同步的实现见csapp相关章节。

##### 互斥锁和条件变量同步

{% highlight c++ linenos %}
    void sbuf_insert(sbuf_t *sp, int item)
    {
        /*write to the buffer*/
        Pthead_mutex_lock(&sp->buf_mutex);
        if(sp->nslots == 0)
        {
            Pthread_mutex_unlock(&sp->buf_mutex);
            return ;
        }
        sp->buf[(++sp->rear)%(sp->n)] = item;
        sp->nslots--;
        Pthread_mutex_unlock(&sp->buf_mutex);

        int dosignal = 0;
        Pthread_mutex_lock(&sp->nready_mutex);
        if(sp->nready == 0)
            dosignal = 1;
        sp->nready++;
        Pthread_mutex_unlock(&sp->nready_mutex)
        if(dosignal)
            Pthread_cond_signal(&sp->cond);
    }
    int sbuf_remove(sbuf_t *sp)
    {
        int item;
        Pthread_mutex_lock(&sp->nready_mutex);
        while(sp->nready == 0)
            Pthread_cond_wait(&sp->cond,&sp->nready_mutex);
        item = sp->buf[(++sp->front) % (sp->n)];
        Pthread_mutex_unlock(&sp->nready_mutex);
        if(item == 0)fprintf(stderr, "error!!!!fd item%d\n", item);
        return item;
    }
{% endhighlight  %}
这个版本，主函数与版本3一致，缓冲区的同步我改用了互斥锁和条件变量。这里贴出sbuf insert和remove操作的实现。其中sbuf_t结构体中，nready和nslots分别指准备好待消费的描述符和缓冲区剩余的空位。

在这里，为什么在需要两个同步变量nready和nslots对应两个互斥锁？

任意使用其中一个，当nslots小于n,或者nready大于零的时候，唤醒等待在条件变量上的线程，这样只需用一个同步变量，详见源码。我自己测试了一下，两种方式效率是差不多的。

我的理解是，当使用两个同步变量时，生产者在放入产品的时候，不阻塞消费者消费其他产品，因为没有对nready加锁，所以如果第一个阶段（放入产品）耗时比较多时，用两个同步变量更合适一些。而这里，放入产品并不是耗时操作，因此效率差不多。

还有一个需要注意的地方是，我把Pthread_cond_signal放到了mutex外面，是为了避免上锁冲突，见UNP卷2 7.5。

##### 线程各自accept
{% highlight c++ linenos %}
    int i;
    for(i=0;i<NTHREADS;i++)/*create worker threads*/
        Pthread_create(&tid,NULL,thread,NULL);
    while (1) {
        pause();
      }
    }
    /* $end tinymain */  
    void *thread(void *vargp)
    {
      Pthread_detach(pthread_self());
      int connfd;
      struct sockaddr_in clientaddr;
      int clientlen = sizeof(clientaddr);

      while(1)
      {
        Pthread_mutex_lock(&thead_lock);
        connfd = Accept(listenfd, (SA *)&clientaddr,&clientlen);
        Pthread_mutex_unlock(&thead_lock);
        doit(connfd);
        Close(connfd);
      }
    }
{% endhighlight %}
这个版本是预先创建固定数量的，但是由线程各自去accept, 对accept上锁保护。这种方式显然代码实现上容易得多，在效率上，由于只用到了Pthread_mutex_lock这一种系统调用，效率应该要稍微好一点。

以上就是我实现过的基于进程、线程，主要是线程的并发模式。进程另外还有几种模式，我认为那几种模式和线程基本一致，代码写起来也比较类似。实际上，在Linux下，线程其实就是资源共享的进程，都有自己的task_struct结构（《Linux内核设计与实现》）。

## 二、基于I/O多路复用的并发

在这一部分，主要介绍linux下面，select、poll和epoll的用法和示例。一共下面三个程序：

- tiny_select
- tiny_poll
- tiny_epoll_nonblock 非阻塞I/O模式

### select函数

{% highlight c++ linenos %}
    typedef struct 
    {
        int maxfd;
        fd_set read_set; 
        fd_set ready_set;
        int nready;
        int maxi;
        int clientfd[FD_SETSIZE];
    } pool;
    static pool client_pool;
    init_pool(listenfd,&client_pool);
    while(1)
    {
    	client_pool.ready_set = client_pool.read_set;
    	while((client_pool.nready = Select(client_pool.maxfd+1,&client_pool.ready_set,NULL,NULL,NULL)) < 0)
    	{
            if(errno == EINTR)
                printf("got a signal restart pselect!!! \n");
    	}
    	/*mask SIGCHLD!!!!!    but some signal will be abondoned
    	*/
    	//client_pool.nready = Pselect(client_pool.maxfd+1,&client_pool.ready_set,NULL,NULL,NULL,&sig_chld);
    	if(FD_ISSET(listenfd,&client_pool.ready_set)){
    	    connfd = Accept(listenfd,(SA *)&clientaddr,&clientlen);
    		add_client(connfd,&client_pool);
    	} 
    	check_clients(&client_pool);
    }
{% endhighlight %}

第一个版本，来自于csapp。在pool结构体中clientfd保存所有已连接的fd，read_set是需要select去检测的fd集,ready是select返回已经准备好的fd集。需要注意的地方有：
1. 在调用select函数的地方，用while的原因是，子进程中断的SIGCHLD的信号会导致select返回，需要手动重启。最开始我用了Pselect来解决这个问题，但这样会造成信号的丢失。
2. 在每次调用select之前，需要对ready_set重新赋值，select返回之后，ready_set会被修改，在下次调用之前需要恢复。

再看一下这个client_pool的实现：


{% highlight c++ linenos %}
    void init_pool(int listenfd, pool *p) {
      int i;
      p->maxi = -1;
      for (i = 0; i < FD_SETSIZE; i++) p->clientfd[i] = -1;

      p->maxfd = listenfd;
      FD_ZERO(&(p->read_set));
      FD_SET(listenfd, &p->read_set);
    }

    void add_client(int connfd, pool *p) {
      int i;
      p->nready--;
      for (i = 0; i < FD_SETSIZE; i++) {
        if (p->clientfd[i] < 0) {
          p->clientfd[i] = connfd;
          FD_SET(connfd, &p->read_set);

          if (connfd > p->maxfd) p->maxfd = connfd;
          if (i > p->maxi) p->maxi = i;   
          break;
        }
      }
      if (i == FD_SETSIZE) app_error("add_client error: Too many clients");
    }

    void check_clients(pool *p) {
      int i, connfd, n;

      for (i = 0; (i <= p->maxi) && (p->nready > 0); i++) {
        connfd = p->clientfd[i];
        if ((connfd > 0) && (FD_ISSET(connfd, &p->ready_set))) {
          p->nready--;
          doit(connfd);
          Close(connfd);
          FD_CLR(connfd, &p->read_set);
          p->clientfd[i] = -1;
        }
      }
    }
{% endhighlight %}

init_pool初始化，最开始的fd_set里面只有listenfd，add_client将已连接描述符添加到clientfd,并将read_set置位。check_clients是循环依次检查是哪一个已连接fd,处理完毕后，将fd从clientfd和read_set中移除。

从上面的过程中，可以看出select有几个明显的缺点：
- 第一个是fd_set是固定大小的，最大为FD_SETSIZE（修改起来比较麻烦），其实就是一个数组，这是极大的一个限制。
- 第二个是我们在每次调用select时，实际上需要把所有的连接描述符从用户空间拷贝到内核空间，内核检测到准备好的描述符过后，将结果集再拷贝回来。这样的拷贝，当你fd的数量很多的时候，消耗是非常大的。
- 第三个是select在内核中，是通过轮询遍历所有fd,这种开销也是很大的。具体可以参考select有关的内核源码。

### poll函数

{% highlight c++ linenos %}
    struct pollfd{
        int fd; //fd to check
        short events; //events of interest on fd
        short revents; //events that occurred on fd
    }
    typedef struct {
        struct pollfd client[OPEN_MAX];
        int maxi;
        int nready;
    } pool;
    static pool client_pool;
    init_pool(listenfd, &client_pool);
    while (1) {
        while ((client_pool.nready =
            Poll(client_pool.client, client_pool.maxi + 1, INFTIM)) < 0) {
        if (errno == EINTR) printf("got a signal restart poll!!! \n");
        }
        if (client_pool.client[0].revents & POLLRDNORM) {
            connfd = Accept(listenfd, (SA *)&clientaddr, &clientlen);
            add_client(connfd, &client_pool);
        }
        check_clients(&client_pool);
    }
{% endhighlight %}

poll的代码，基本与select模式一致。poll不同的地方在于它使用pollfd结构来表示fdset，而不是位图。没有大小限制，使用起来更为方便。events和revents分别表示需要检测的事件和发生的事件。poll传入的client指针，底层应该是所有的pollfd构成的一个链表。

poll和select的缺点一样，都需要拷贝和轮询，随着fd数量的增大，效率都会大大降低。

### epoll+非阻塞IO

{% highlight c++ linenos %}
    typedef struct request_buffer{
        int fd;/*fd for the connection  */
        int epfd;/* fd for epoll */
        char buf[MAXBUF]; /*the buffer for the current request*/
        size_t pos,last;
    }request_b
    struct epoll_event event;  // event to register
    event.data.ptr = (void *)request;
    event.events = EPOLLIN | EPOLLET;
    Epoll_ctl(epfd, EPOLL_CTL_ADD, listenfd, &event);

    while (1) {
        int n;
        while ((n = Epoll_wait(epfd, events, MAXEVENTS, -1)) < 0) {
        if (errno == EINTR) printf("got a signal restart\n");
        }
        for (int i = 0; i < n; i++) {
        request_b *rb = (request_b *)events[i].data.ptr;
        // int fd = rb->fd;
        if ((events[i].events & EPOLLERR) || (events[i].events & EPOLLHUP) ||
            (!(events[i].events & EPOLLIN))) {
            fprintf(stderr, "epoll error fd %d", rb->fd);
            Close(rb->fd);
            continue;
        } else if (rb->fd == listenfd) {
            /* the new connection incoming */
            int infd;
            while (1) {
            infd = accept(listenfd, (SA *)&clientaddr, &clientlen);
            if (infd < 0) {
                if ((errno == EAGAIN) || (errno == EWOULDBLOCK)) {
                /*we have processed all incoming connections*/
                break;
                } else {
                unix_error("accept error!");
                break;
                }
            }
            make_socket_non_blocking(infd);
            if (verbose) printf("the new connection fd :%d\n", infd);
            request_b *request = (request_b *)Malloc(sizeof(request_b));
            request_init(request, infd, epfd);
            event.data.ptr = (void *)request;
            event.events = EPOLLIN | EPOLLET | EPOLLONESHOT;
            Epoll_ctl(epfd, EPOLL_CTL_ADD, infd, &event);
            }
        }
        
        else {
            if (verbose) printf("new data from fd %d\n", rb->fd);
            doit_nonblock((request_b *)events[i].data.ptr);
            /* where to close the fd and release the requst!!! */
        }
        }
    }
{% endhighlight %}

这段代码是典型的non-blocking IO + IO multiplexing的模式，由于是non-blocking IO,就不能用前面csapp提供的Rio包，因为要处理数据分包到达的情况。可以注意到结构体request_b就是是用来记录当前fd对应的请求状态，buf是当前请求的缓冲区。doit_nonblock和前面的doit函数有了很大的变化，这里我就不展开了，可以看我的代码，待完善。

epoll有ET和LT两种模式，详细定义见man手册。ET模式能够减少事件触发的次数，但是代码复杂度会增加，IO操作必须要等到EAGAIN，容易漏掉事件。而LT模式，事件触发的次数多一些，代码实现上要简单一点，不容易出错。目前没有明确的结论哪种模式更高效，应该是看具体的场景。我这里使用了ET模式，在我的doit_nonblock函数里面，对于请求结束的判断还有错误，不能够正确判断一个完整的HTTP请求。

前面说到select的缺点，epoll是怎么解决的呢？
- epoll是在epoll_ctl的时候，就把当前fd"注册"到内核中，而不是在epoll_wait的时候,时效更长，避免了大量重复的拷贝。
- epoll内部采用红黑树来组织所有fd（结构体epitem）,用一个双向链表存储发生事件的fd（结构体epitem）。epoll_wait只需要检查这个双向链表是否为空来判断是否有事件发生。
- epoll在执行epoll_ctl的时候，除把fd了插入红黑树之外,还会给内核中断处理程序注册回调函数。在这个回调函数里面，会把准备好的fd插入就绪的双向链表里面。

因此，当存在有大量的连接，但是活跃连接数量较少的情况下，epoll是十分高效的。更多的细节，可以参考内核源码


## 三、总结
在实际使用中，肯定不会单纯的用上面的某一种模式，而是多进程+IO multiplexing 或者 多线程 + IO multiplexing。后面再结合具体的例子，我会再继续研究。原本，我写了一个小的测试程序，但是发现我的测试方法不是十分合理，没有太大意义，就没有放出来了，有兴趣的可以看一看源码里面。

我是从读csapp后半部分开始，集中学习了一下网络编程的内容，这些内容十分基础，也许会对初学者有一些帮助。后续，我还会继续深入，准备阅读陈硕的muduo，自己再动手写一些代码。



参考链接：

- https://segmentfault.com/a/1190000003063859
- http://lifeofzjs.com/blog/2015/05/16/how-to-write-a-server/
- https://banu.com/blog/2/how-to-use-epoll-a-complete-example-in-c/
- http://www.cnblogs.com/apprentice89/p/3234677.html