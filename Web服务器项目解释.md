## 	Web服务器项目解释

### 1、线程池

```c++
#ifndef THREADPOOL_H
#define THREADPOOL_H

#include <mutex>
#include <condition_variable>
#include <queue>
#include <thread>
#include <functional>

//线程池就是一个生产者与消费者模型，当有任务发生的时候就往任务队列添加任务，主线程类似于生产者，子线程类似于消费者
class ThreadPool {
public:
    //构造函数，进行初始化，初始化的时候线程池中的线程的数量初始化为8，新建一个池子
    explicit ThreadPool(size_t threadCount = 8): pool_(std::make_shared<Pool>()) {  //explict修饰i构造函数时，可以防止隐式转换和复制初始化
            assert(threadCount > 0);  //断言，确保不发生错误

            // 创建threadCount个子线程
            for(size_t i = 0; i < threadCount; i++) {
                std::thread([pool = pool_] {  //创建一个线程，这里根据上述的循环，每循环一次创建一个线程，直到创建完八个线程就跳出循环
                    std::unique_lock<std::mutex> locker(pool->mtx);  //创建互斥锁，这里每次创建一个线程都会去查看任务队列是否又任务到来，有的话就挑一个线程去处理任务
                    while(true) {  //这里一直循环的原因是因为线程需要不停的去查看任务队列是否有任务到来
                        if(!pool->tasks.empty()) {  //当任务不为空的时候
                            // 从任务队列中取第一个任务
                            auto task = std::move(pool->tasks.front());
                            // 移除掉队列中第一个元素
                            pool->tasks.pop();
                            locker.unlock();  //解锁
                            task();  //任务执行的代码
                            locker.lock();//加锁
                        } 
                        //任务的队列不为空的时候
                        else if(pool->isClosed) break;  //此时线程池是关闭的
                        else pool->cond.wait(locker);   // 如果队列为空，等待，阻塞在这里，当往队列添加任务时就可以被唤醒继续执行，利用一个信号量来通知线程任务的到来。
                    }
                }).detach();// 线程分离，主要作用是为了线程退出的时候不需要主线程去回收它的资源，自己释放即可
            }
    }

    ThreadPool() = default;  //无参构造函数

    ThreadPool(ThreadPool&&) = default;  //拷贝构造函数
    
    ~ThreadPool() {  //析构函数
        if(static_cast<bool>(pool_)) {
            {
                std::lock_guard<std::mutex> locker(pool_->mtx);  //创建互斥锁
                pool_->isClosed = true;  //把当前是否关闭设置为true，表示要关闭
            }
            pool_->cond.notify_all();  //条件变量通知   //通过通知
        }
    }

    template<class F>
    void AddTask(F&& task) {  //往队列中添加新的任务
        {
            std::lock_guard<std::mutex> locker(pool_->mtx);  //先上锁
            pool_->tasks.emplace(std::forward<F>(task));  //添加任务
        }
        pool_->cond.notify_one();   // 唤醒一个等待的线程
    }

private:
    // 结构体，池子，定义一个pool类型的结构体，结构体的属性包含互斥锁、条件变量、是否关闭以及一个任务队列，任务队列的作用就是为了保存产生的任务
    struct Pool {
        std::mutex mtx;     // 互斥锁
        std::condition_variable cond;   // 条件变量
        bool isClosed;          // 是否关闭
        std::queue<std::function<void()>> tasks;    // 队列（保存的是任务）
    };
    std::shared_ptr<Pool> pool_;  //  池子，根据上面的结构体类型创建的池子
};


#endif //THREADPOOL_H
```



解释：线程池：就是里面创造了多个线程，主要的目的是因为如果有任务到来的话，我们每次一个任务到来都创建一个线程的话这样很耗时，而且浪费资源，当线程结束以后还需要去释放线程，而线程池的作用就是预先创建一些线程放到一个池子里，每次有任务到来的时候就从线程池中取一个线程去处理，处理完任务以后，线程不需要被释放，而是重新放入线程池里面，等到后面的任务到来的时候再用来处理任务。

进程：就是运行的程序，进程是资源分配的主要单位，计算机的资源的分配都是以进程为单位，例如电脑上的qq、浏览器、音乐播放器等这些都属于进程，进程的创建同样也要消耗计算机的资源，进程在创建的时候是相互独立的，大部分的资源都是不共享的，所以进程一般都是读的时候共享资源，需要写的时候就需要复制相同的资源，例如内存中的一块存储了一个值a=10，当两个进程都去读的时候都不会改变值得大小，但是如果两个进程都去写得话，例如进程1要将a的值改成30，进程2需要将a的值改成-10，这样就会发生矛盾，所以需要各自复制一份内存中的值去自己玩，如果有很多进程的话，那就需要复制无数份，很占内存。

线程：线程是计算机执行的基本单位，线程是轻量级的进程，它大部分的时候都是共享同一份资源，在多个线程对资源进行操作的时候，需要采用同步和互斥的方式，主要的方法就是互斥锁、条件变量和信号量

互斥锁：当我们多个线程对同一个变量进行操作的时候，我们需要定义一个互斥锁，即一个线程去访问资源的时候，先把资源锁上，据为己有，其他线程也想要访问该资源的时候，一看被锁上那就只能排队等候，类似于生活中的公共厕所，但是只有一个公共厕所，大家都需要使用这个公共厕所。当一个线程在访问资源的时候（一个人进了公共厕所），那其他的线程就需要在外面等待，因为访问资源的线程将资源锁上（一个人进去上厕所先锁门），等第一个线程访问完资源后，它释放锁（一个人上完厕所开门），其他的线程就可以按序进行访问，这就是互斥锁的原理。

条件变量：条件变量的的解释就是发生了某一条件需要通知其他的线程，例如银行的业务办理，假设只有一个窗口可以办理业务，要办理业务的人就是线程，那么去银行都会预先取号，大家就不需要排队，到休息区等待叫号，叫号就是一个条件变量的机制。在上述的代码中条件变量在三个地方用到，一个是线程去任务队列中取任务的时候，如果没有任务，那么就阻塞在那里等待，当添加任务的时候，就通知线程有任务到来，还有一个地方就是析构函数那里关闭池子也是通过条件变量通知所有的线程。

![image-20210713091834522](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210713091834522.png)

### 2、epoller--IO多路复用技术

![image-20210714101916317](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210714101916317.png)

头文件

```c++
#ifndef EPOLLER_H
#define EPOLLER_H

#include <sys/epoll.h> //epoll_ctl()
#include <fcntl.h>  // fcntl()
#include <unistd.h> // close()
#include <assert.h> // close()
#include <vector>
#include <errno.h>

class Epoller {
public:
    explicit Epoller(int maxEvent = 1024);  //构造函数，参数是表示最多的事件的数量

    ~Epoller();

    bool AddFd(int fd, uint32_t events);  //往epoll添加要检测的事件
 
    bool ModFd(int fd, uint32_t events);  //修改epoll里面的要检测的事件

    bool DelFd(int fd);  //删除要检测的事件

    int Wait(int timeoutMs = -1);  //等待检测，让内核去检测

    int GetEventFd(size_t i) const;  //获取事件的fd

    uint32_t GetEvents(size_t i) const;  
        
private:
    int epollFd_;   // epoll_create()创建一个epoll对象，返回值就是epollFd

    std::vector<struct epoll_event> events_;     // 检测到的事件的集合 
};

#endif //EPOLLER_H
```

epoller.cpp文件

```c++
#include "epoller.h"

// 创建epoll对象 epoll_create(512)
Epoller::Epoller(int maxEvent):epollFd_(epoll_create(512)), events_(maxEvent){  //events_表示返回的事件的集合
    assert(epollFd_ >= 0 && events_.size() > 0);
}

Epoller::~Epoller() {
    close(epollFd_);
}

// 添加文件描述符到epoll中进行管理
bool Epoller::AddFd(int fd, uint32_t events) {
    if(fd < 0) return false;
    epoll_event ev = {0};  //开始的时候将事件初始化为0
    /*
    struct epoll_event
    {
        uint32_t events;	// Epoll events 
        epoll_data_t data;	// User data variable
    }__EPOLL_PACKED;


    typedef union epoll_data
    {
        void *ptr;
        int fd;
        uint32_t u32;
        uint64_t u64;
    } epoll_data_t;


    */
    ev.data.fd = fd;
    ev.events = events;
    return 0 == epoll_ctl(epollFd_, EPOLL_CTL_ADD, fd, &ev);  //往内添加事件，添加成功返回0
}

// 修改
bool Epoller::ModFd(int fd, uint32_t events) {
    if(fd < 0) return false;
    epoll_event ev = {0};
    ev.data.fd = fd;
    ev.events = events;
    return 0 == epoll_ctl(epollFd_, EPOLL_CTL_MOD, fd, &ev);  //修改事件，修改成功返回0
}

// 删除
bool Epoller::DelFd(int fd) {
    if(fd < 0) return false;
    epoll_event ev = {0};
    return 0 == epoll_ctl(epollFd_, EPOLL_CTL_DEL, fd, &ev);//删除成功返回0
}

// 调用epoll_wait()进行事件检测
int Epoller::Wait(int timeoutMs) {
    return epoll_wait(epollFd_, &events_[0], static_cast<int>(events_.size()), timeoutMs);  //timeouts是阻塞时间，该函数成功返回的是变化的文件描述符的个数
}

//当内核程序向用户返回信息的时候，不仅需要返回的是内核检测到的文件描述符的信息，同时还需要返回的是该文件描述符发生的事件的类型，具体是哪一类事件，
//所以需要下面的两个函数

// 获取产生事件的文件描述符
int Epoller::GetEventFd(size_t i) const {
    assert(i < events_.size() && i >= 0);
    return events_[i].data.fd;   //返回的是发生变化的文件描述符
}

// 获取事件
uint32_t Epoller::GetEvents(size_t i) const {
    assert(i < events_.size() && i >= 0);
    return events_[i].events;  //获取发生变化的文件描述符的事件
}
```

**解释**

IO多路复用技术：

我们小学曾经做过一个实验是用纸杯进行通话，我们两个人相互通话需要的时候需要两个纸杯，三个人通话的时候按照惯例是需要的纸杯的数量是大于三的，但是我们可以通过复用技术来实现，只需要三个就可以通话。同理对于网络通信来说，如果每次发生一个事件都需要创建一个套接字去监听，那么如果通信的用户的数量太多的时候需要我们创建的套接字就很多，所以我们可以通过IO复用技术来实现。

原始的网络编程打监听方法：

![image-20210714105431554](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210714105431554.png)

上图的上面部分表示的是还没有驿站存在的情况，我们的客户和快递员直接取得联系，有一个快递就需要一个快递员去送货，并且用户还要去一个一个的签收快递。

**IO复用技术的主要三种方法：**

**1、select**

![image-20210714105642136](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210714105642136.png)

select的原理如上图所示，每个客户只需要告诉快递代收点，我有哪些快递到了，当快递到来的时候，快递员只告诉你有几个快递到了，但是具体哪几个需要我们依次去遍历。同理，对于发生的事件，内核和用户的交流也是如此，如下图所示：

![image-20210714111540340](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210714111540340.png)

**存在的缺点：**

1、每次遍历的时候都需要将上面的集合从用户区拷贝到内核，这样会使得计算机的开销很大

2、每次在内核区域进行监听的时候也是需要0到1023一个一个的去监听哪一个发生了事件，开销也很大

3、每次从内核将发生变化后的集合返回给用户区，也是开销大，同时用户还需要一个一个的去找哪一个文件描述符发生了变化

4、用户初始创建的集合不能够重用，每次都需要复制一个副本传进去内核，所以开销大

5、select的文件描符的数量只能是1024个，有限制



**2、poll**

poll和select的方法差不多，区别主要在于poll定义了一个结构体类型的实例，

![image-20210714122013153](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210714122013153.png)

这样的话用户每次传入内核的数据只是委托内核检测的文件描述符和描述符的具体事件，而内核修改的是文件描述符实际发生的事件，这样的话就不需要每次都要复制一个副本，所以更加的方便。

**3、epoll**

![image-20210714122813627](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210714122813627.png)

![image-20210714122237609](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210714122237609.png)

epoll的工作流程如上图所示，如上述代码部分的实现原理，首先在内核创建一个epoll事件，主包含两部分，一部分是一个红黑树的数组，主要记录的是需要监听的文件描述符，采用红黑树的效率非常高，同时一个双链表记录的实际发生变化的事件类型，返回给用户的时候只需要返回发生变化的文件描述符即可，这样的话效率也很高，开销小，所以优势很大。

**epoll的两种工作模式**

![image-20210714122909271](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210714122909271.png)

![image-20210714122939402](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210714122939402.png)

**epoll的oneshot事件**

即使我们使用的是ET模式的高效模式，但是当某一个套接字发生事件时候，线程池会选出一个线程进行处理，当该线程在处理这个任务的时候，这个套接字又发生了事件，那么线程池又会重新来一个线程对事件进行处理，这样的话就会发生两个线程处理同一个套接字的情况，而我们期望的是一个套接字只能被一个线程处理，，所以需要使用epoll的oneshot事件来实现，使得一个套接字在被某一个线程处理的时候，其他的线程是无法来处理的。

### 3、自动增长的缓冲区的实现原理

在对文件描述符的数据进行处理的时候，因为要先把数据读取到缓冲区中，再从缓冲区读取出来进行处理，所以缓冲区的大小影响着我们对数据的读取，下面是它的实现原理

首先我们再内存中创建一个缓冲区，缓冲区的大小为1024字节，同时定义两个指针，一个读指针，表示的是可以读取的数据的开始位置，一个写指针表示的是可以写入数据的起始位置，开始的时候两根指针都是指向内存开始的地方，如下图所示：

![image-20210715093649496](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210715093649496.png)

当有文件描述符监听到事件的发生的时候，比如发生的是读事件，那么就要把文件描述符监听到的事件读取到缓冲区，加入读取的字节的长度为len，那么写指针的位置就要发生变化，如下图所示：

![](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210715094146560.png)

因为缓冲区的大小是固定的大小，而我们通常是一次性将数据全部读取到缓冲区，那么就有可能装不下数据，所以需要临时创建一个缓冲区来缓解，将存不下的放到临时缓冲区，这样就可以一次性将所有的数据读入，这里利用临时缓冲区的技术是一个分散读的技术，即将数据分散读取到内存中不同的位置，分散读的话需要定义一个结构体数组如下所示：

```c++
struct iovec
{
	void * iov_base;  //表示的是缓冲的地址
	size_t iov_len;  //表示的是缓冲的大小
}
```

生成的结构体数组如下图所示：

![image-20210715095810663](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210715095810663.png)

如上图所示，当固定的缓冲区想要继续写数据的时候，发现剩余的位置不够写的时候那么就可以先把数据写入到临时缓冲中，再将临时数组的数据读取到固定缓冲区来处理，比如上图所示读指针的位置表示前面的数据已经被读取了，写指针的位置表示如果有数据那么要从这里开始写。那么如何实现动态扩容的呢？

**因为数据的处理只能放到固定大小的缓冲中进行处理，即上述的1024字节的缓冲中，那么如果很多数据都读取到这块区域的话，那么肯定是放不下的，我们就可以利用一个临时的缓冲区，把放不下的数据先放到临时的缓冲的位置，等到1024字节大小的内存有剩余的空间的时候我再将临时缓冲区的数据放入到1024的位置进行处理，那么何时1024缓冲中有空闲的位置呢？**原理及实现如下图所示：

![image-20210715101824831](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210715101824831.png)

可以从上图看出，如果我们想要写入数据，那么此时可以利用的空间就是最前面的部分和最后面的部分的位置，但是写入数据一定要连续，所以唯一的办法就是将中间的数据移动到最前面，这样就可以将空闲的区域连接在一起，方便后面的写数据。具体的实现就是将读指针到写指针之间的数据复制到最前面，再更改读指针和写指针的位置，这就是利用一个缓冲实现自动增长的原理，关键部分的代码如下：

这是往缓冲中写入数据：

![image-20210715103121366](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210715103121366.png)

往缓冲读取数据

![image-20210715103210547](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210715103210547.png)

动态扩容的实现：

![image-20210715103235156](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210715103235156.png)

### 4、资源的请求与解析

#### 有限状态机

逻辑单元内部的一种高效编程方法：有限状态机（finite state machine）。
有的应用层协议头部包含数据包类型字段，每种类型可以映射为逻辑单元的一种执行状态，服务器可以
根据它来编写相应的处理逻辑。如下是一种状态独立的有限状态机：

```c++
STATE_MACHINE( Package _pack )
{
	PackageType _type = _pack.GetType();
	switch( _type )
	{
		case type_A:
			process_package_A( _pack );
			break;
		case type_B:
			process_package_B( _pack );
			break;
	}
}
```

这是一个简单的有限状态机，只不过该状态机的每个状态都是相互独立的，即状态之间没有相互转移。
状态之间的转移是需要状态机内部驱动，如下代码：

![image-20210726100812434](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210726100812434.png)

该状态机包含三种状态：type_A、type_B 和 type_C，其中 type_A 是状态机的开始状态，type_C 是状
态机的结束状态。状态机的当前状态记录在 cur_State 变量中。在一趟循环过程中，状态机先通过
getNewPackage 方法获得一个新的数据包，然后根据 cur_State 变量的值判断如何处理该数据包。数据
包处理完之后，状态机通过给 cur_State 变量传递目标状态值来实现状态转移。那么当状态机进入下一
趟循环时，它将执行新的状态对应的逻辑。

#### 资源的请求

每个客户端连接进来以后，都是通过创建一个HttpConn连接的对象，加入到任务队列中，然后再对队列进行逻辑处理

**1、构造函数定义一系列的请求的状态信息和Http的状态码**

![image-20210726104410952](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210726104410952.png)

首先的时候，每个请求对象连接进来都是封装为一个对象，存储在任务队列中

![image-20210726104657465](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210726104657465.png)

上面这符图里面实现的是状态的转换，即开始的时候都是解析请求首行，根据解析请求是否完成，执行下一个状态，利用的就是有限状态机的思想

（1）、解析请求首行

![image-20210726104921337](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210726104921337.png)

（2）、解析请求头

![image-20210726105106278](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210726105106278.png)

（3）、解析请求体

![image-20210726105127863](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210726105127863.png)

如果请求是Post请求的话，还需要进行如下操作

![image-20210726105212423](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210726105212423.png)

验证用户的登录信息

![image-20210726105315145](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210726105315145.png)



![image-20210726105327288](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210726105327288.png)

#### 响应信息

![image-20210726110538681](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210726110538681.png)





### 5、定时器的设计

定时是指在一段时间之后触发某段代码的机制，我们可以在这段代码中依次处理所有到期的定时器，定时器的实现是基于小根堆来实现的，结点的插入、删除、添加与堆的操作一样

因为定时器的排序是按照根结点从大到小的顺序排序的，所以当某个结点有新的数据到来或者需要读取新的数据的时候就需要调整该结点的定时器的大小，然后重新排序后保证定时器的大小还是按照从小到大的顺序去排列。

1、调整结点的代码

![image-20210726135813868](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210726135813868.png)

先计算当前的超时时间的大小，然后通过对比，依次向下调整

2、添加超时时间

![image-20210726140006186](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210726140006186.png)

先判断结点是否为新的结点，如果是新的结点的话，那就插入到堆的最后一个位置，然后依次向上调整，不是新的结点的话，那就按照上面的调整的方法来解决。

3、删除指定的结点

![image-20210726140153578](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210726140153578.png)

![image-20210726140224058](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210726140224058.png)

删除元素的操作与堆的删除元素的操作一样，先将要删除的元素的结点与最后一个结点的位置进行交换，然后删除最后一个位置的元素，删除完毕以后再调整堆。

4、清除超时的结点

![image-20210726140432585](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210726140432585.png)

先比较当前的结点的定时器的值与系统当前的时间的大小，然后决定是否将定时器删除

5、获取下一个位置的定时器信息并比较

![image-20210726140536889](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210726140536889.png)

### 6、数据库的连接池的实现

数据库的连接池的实现和线程池的实现类似，都是封装为对象然后放到池中，有客户端请求数据的时候，挑选出一个数据库的对象为相应的请求服务。

连接的实例是一个静态变量，里面也用到了相关的操作：互斥锁、队列、信号量等

![image-20210726143518955](C:\Users\tz\AppData\Roaming\Typora\typora-user-images\image-20210726143518955.png)

```bash
.
├── LICENSE
├── Mywebserver
│   ├── bin
│   │   └── server
│   ├── build
│   │   └── Makefile
│   ├── code
│   │   ├── buffer
│   │   │   ├── buffer.cpp
│   │   │   └── buffer.h
│   │   ├── config
│   │   │   └── config.h
│   │   ├── http
│   │   │   ├── httpconn.cpp
│   │   │   ├── httpconn.h
│   │   │   ├── httprequest.cpp
│   │   │   ├── httprequest.h
│   │   │   ├── httpresponse.cpp
│   │   │   └── httpresponse.h
│   │   ├── log
│   │   │   ├── blockqueue.h
│   │   │   ├── log.cpp
│   │   │   └── log.h
│   │   ├── main.cpp
│   │   ├── pool
│   │   │   ├── sqlconnpool.cpp
│   │   │   ├── sqlconnpool.h
│   │   │   ├── sqlconnRAII.h
│   │   │   └── threadpool.h
│   │   ├── server
│   │   │   ├── epoller.cpp
│   │   │   ├── epoller.h
│   │   │   ├── webserver.cpp
│   │   │   └── webserver.h
│   │   └── timer
│   │       ├── heaptimer.cpp
│   │       └── heaptimer.h
│   ├── LICENSE
│   ├── log
│   │   └── 2021_07_26.log
│   ├── Makefile
│   ├── readme.assest
│   │   └── 压力测试.png
│   └── resources
│       ├── 400.html
│       ├── 403.html
│       ├── 404.html
│       ├── 405.html
│       ├── css
│       │   ├── animate.css
│       │   ├── bootstrap.min.css
│       │   ├── font-awesome.min.css
│       │   ├── magnific-popup.css
│       │   └── style.css
│       ├── error.html
│       ├── fonts
│       │   ├── FontAwesome.otf
│       │   ├── fontawesome-webfont.eot
│       │   ├── fontawesome-webfont.svg
│       │   ├── fontawesome-webfont.ttf
│       │   ├── fontawesome-webfont.woff
│       │   └── fontawesome-webfont.woff2
│       ├── images
│       │   ├── favicon.ico
│       │   ├── instagram-image1.jpg
│       │   ├── instagram-image2.jpg
│       │   ├── instagram-image3.jpg
│       │   ├── instagram-image4.jpg
│       │   ├── instagram-image5.jpg
│       │   └── profile-image.jpg
│       ├── index.html
│       ├── js
│       │   ├── bootstrap.min.js
│       │   ├── custom.js
│       │   ├── jquery.js
│       │   ├── jquery.magnific-popup.min.js
│       │   ├── magnific-popup-options.js
│       │   ├── smoothscroll.js
│       │   └── wow.min.js
│       ├── login.html
│       ├── picture.html
│       └── register.html
└── README.md

```

