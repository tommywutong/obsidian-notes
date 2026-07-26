---
title: "【iOS】多线程与GCD"
slug: "ios-gcd"
published: 2025-11-17
description: "线程与进程 线程和进程的定义 线程 线程是进程的基本执行单元，一个进程的所有任务都在线程中执行 进程要想执行任务，必须要有线程，进程至少要有一条线程 程序启动时会默认开启一条线程，这条线程被称为主线程或UI线程 进程 进程是指在系统中正在运行的一个应用程序 每个进程之间是独立的，"
tags: ["iOS", "GCD", "多线程"]
category: "iOS"
draft: false
---

## 线程与进程


### 线程和进程的定义


#### 线程


- 线程是进程的基本执行单元，一个进程的所有任务都在线程中执行
- 进程要想执行任务，必须要有线程，进程至少要有一条线程
- 程序启动时会默认开启一条线程，这条线程被称为主线程或UI线程


#### 进程


- 进程是指在系统中正在运行的一个应用程序
- 每个进程之间是独立的，每个进程均运行在其专用的且受保护的内存空间内
- 通过“活动监视器”可以查看mac系统内开启的线程 所以，可以简单理解为：进程是线程的容器，而线程用来执行任务。在iOS中是单进程开发，一个进程就是一个app，进程之间是相互独立的，如支付宝、微信、qq等，这些都是属于不同的进程


#### 进程与线程的关系


进程与线程之间的关系主要涉及两个方面：


- 地址空间 同一个进程的线程共享本进程的地址空间
- 而进程之间则是独立的地址空间


- 资源拥有 同一个进程内线程共享本进程的资源，如内存、I/O、cpu等
- 但是进程之间资源是独立的 两个之间的关系就相当于工厂与流水线的关系，工厂与工厂之间是相互独立的，而工厂中的流水线是共享工厂的资源的，即 进程相当于一个工厂，线程相当于工厂中的一条流水线 针对进程和线程，还有以下几点说明：


- 1: 多进程要比多线程健壮 一个进程崩溃后，在保护模式下不会对其他进程产生影响
- 而一个线程崩溃整个进 程都死掉


- 2: 使用场景：频繁切换、并发操作 进程切换时，消耗的资源大，效率高。所以涉及到频繁的切换时，使用线程要好于进程。
- 同样如果要求同时进行并且又要共享某些变量的并发操作，只能用线程不能用进程


- 3: 执行过程 每个独立的进程有一个程序运行的入口、顺序执行序列和程序入口
- 但是 线程不能独立执行，必须依存在应用程序中，由应用程序提供多个线程执行控制。


- 4: 线程是处理器调度的基本单位，但是进程不是。
- 5: 线程没有地址空间,线程包含在进程地址空间中


## 多线程


### 多线程原理


- 对于单核CPU， 同一时间，CPU只能处理一条线程，即只有一条线程在工作
- iOS中的多线程同时执行的本质是CPU在多个人物之间直接进行快速的切换 ，由于CPU调度线程的时间足够快，就造成了 多线程同时执行的效果。其中切换到时间间隔就是时间片。


### 多线程意义


#### 优点


- 能适当提高程序的执行效率
- 能适当提高资源利用率，如CPU、内存
- 线程上的任务执行完成后，线程会自动销毁


#### 缺点


- 开启线程需要占用一定的内存空间，默认情况下，每一个线程占用512KB
- 如果开启大量线程，会占用大量的内存空间，降低程序的性能
- 线程越多，CPU在调用线程上的开销就越大
- 程序设计更加复杂，比如线程间的通信，多线程的数据共享


### 多线程的生命周期


新建 - 就绪 - 运行 - 阻塞 - 死亡

线程与进程

线程和进程的定义

线程


- 线程是进程的基本执行单元，一个进程的所有任务都在线程中执行
- 进程要想执行任务，必须要有线程，进程至少要有一条线程
- 程序启动时会默认开启一条线程，这条线程被称为主线程或UI线程 进程
- 进程是指在系统中正在运行的一个应用程序
- 每个进程之间是独立的，每个进程均运行在其专用的且受保护的内存空间内
- 通过“活动监视器”可以查看mac系统内开启的线程 所以，可以简单理解为：进程是线程的容器，而线程用来执行任务。在iOS中是单进程开发，一个进程就是一个app，进程之间是相互独立的，如支付宝、微信、qq等，这些都是属于不同的进程 进程与线程的关系 进程与线程之间的关系主要涉及两个方面：
- 地址空间 同一个进程的线程共享本进程的地址空间
- 而进程之间则是独立的地址空间


- 资源拥有 同一个进程内线程共享本进程的资源，如内存、I/O、cpu等
- 但是进程之间资源是独立的 两个之间的关系就相当于工厂与流水线的关系，工厂与工厂之间是相互独立的，而工厂中的流水线是共享工厂的资源的，即 进程相当于一个工厂，线程相当于工厂中的一条流水线 针对进程和线程，还有以下几点说明：


- 1: 多进程要比多线程健壮 一个进程崩溃后，在保护模式下不会对其他进程产生影响
- 而一个线程崩溃整个进 程都死掉


- 2: 使用场景：频繁切换、并发操作 进程切换时，消耗的资源大，效率高。所以涉及到频繁的切换时，使用线程要好于进程。
- 同样如果要求同时进行并且又要共享某些变量的并发操作，只能用线程不能用进程


- 3: 执行过程 每个独立的进程有一个程序运行的入口、顺序执行序列和程序入口
- 但是 线程不能独立执行，必须依存在应用程序中，由应用程序提供多个线程执行控制。


- 4: 线程是处理器调度的基本单位，但是进程不是。
- 5: 线程没有地址空间,线程包含在进程地址空间中 多线程 多线程原理
- 对于单核CPU， 同一时间，CPU只能处理一条线程，即只有一条线程在工作
- iOS中的多线程同时执行的本质是CPU在多个人物之间直接进行快速的切换 ，由于CPU调度线程的时间足够快，就造成了 多线程同时执行的效果。其中切换到时间间隔就是时间片。 多线程意义 优点
- 能适当提高程序的执行效率
- 能适当提高资源利用率，如CPU、内存
- 线程上的任务执行完成后，线程会自动销毁 缺点
- 开启线程需要占用一定的内存空间，默认情况下，每一个线程占用512KB
- 如果开启大量线程，会占用大量的内存空间，降低程序的性能
- 线程越多，CPU在调用线程上的开销就越大
- 程序设计更加复杂，比如线程间的通信，多线程的数据共享

多线程的生命周期

新建 - 就绪 - 运行 - 阻塞 - 死亡


![在这里插入图片描述](./csdn-154915356-ios-gcd-images/01-b38587e6206048e1980176ca7cddfbdb.png)


- 新建：主要是实例化线程对象
- 就绪：线程对象调用start方法，将线程对象加入可调度线程池，等待CPU的调用，即调用start方法，并不会立即执行，进入就绪状态，需要等待一段时间，经CPU调度后才执行，也就是从就绪状态进入运行状态
- 运行：CPU负责调度可调度线城市中线程的执行，在线程执行完成之前，其状态可能会在就绪和运行之间来回切换，这个变化是由CPU负责，开发人员不能干预。
- 阻塞：当满足某个预定条件时，可以使用休眠，即sleep，或者同步锁，阻塞线程执行。当进入sleep时，会重新将线程加入就绪中。下面关于休眠的时间设置，都是NSThread的 sleepUntilDate: 阻塞当前线程，直到指定的时间为止，即休眠到指定时间
- sleepForTimeInterval: 在给定的时间间隔内休眠线程，即指定休眠时长
- 同步锁：@synchronized(self)：


- 死亡：分为两种情况 正常死亡，即线程执行完毕
- 非正常死亡，即当满足某个条件后，在线程内部（或者主线程中）终止执行（调用exit方法等退出）


简要说明，就是处于运行中的线程拥有一段可以执行的时间（称为时间片），


- 如果时间片用尽，线程就会进入就绪状态队列
- 如果时间片没有用尽，且需要开始等待某事件，就会进入阻塞状态队列
- 等待事件发生后，线程又会重新进入就绪状态队列
- 每当一个线程离开运行，即执行完毕或者强制退出后，会重新从就绪状态队列中选择一个线程继续执行

线程的exit与cancel说明：


- exit：一旦强行终止线程，后续的代码都不会执行
- cancel：取消当前线程，但不能取消正在执行的线程

**线程的优先级越高，是不是意味着任务的执行越快？**

线程自信的快慢，除了要看优先级，还要查看资源的大小（即任务的复杂程度）、以及CPU调度情况。在NSThread中，线程优先级threadPriority已经被服务质量qualityOfService取代。


在早期NSThread中，我们就可以通过threadPriority设置线程的“优先级”。


```objective-c
NSThread *thread = [[NSThread alloc] initWithBlock:^{
    // 执行任务
}];
thread.threadPriority = 0.8; // 设置优先级，取值范围 0.0 ~ 1.0
[thread start];

0.0是优先级最低，1.0优先级最高，默认值大约为0.5
Apple后来引入了 服务质量 概念，用于更智能地管理线程优先级，在NSThread、NSOperation、GCD都支持这种机制，
NSThread *thread = [[NSThread alloc] initWithBlock:^{
    // 执行任务
}];
thread.qualityOfService = NSQualityOfServiceUserInitiated; // 设置服务质量
[thread start];
```


![请添加图片描述](./csdn-154915356-ios-gcd-images/02-7f539a1baada444193bccb4c728c192f.png)


![在这里插入图片描述](./csdn-154915356-ios-gcd-images/03-aeab961a7f3742cd9cd1708c9616bc1c.png)


## 线程池


![在这里插入图片描述](./csdn-154915356-ios-gcd-images/04-0d8c0521b8a24870b0059ce57f82e942.png)


## iOS多线程


iOS中的多线程实现方式，主要有四种：pthread、NSThread、GCD、NSOperation，汇总如图所示


![在这里插入图片描述](./csdn-154915356-ios-gcd-images/05-90b585e12c774155b58fda6a3451a662.png)


```objective-c
// *********1: pthread*********
pthread_t threadId = NULL;//c字符串char *cString = "HelloCode";/**
 pthread_create 创建线程
 参数：
 1. pthread_t：要创建线程的结构体指针，通常开发的时候，如果遇到 C 语言的结构体，类型后缀 `_t / Ref` 结尾
 同时不需要 `*`
 2. 线程的属性，nil(空对象 - OC 使用的) / NULL(空地址，0 C 使用的)
 3. 线程要执行的`函数地址`
 void *: 返回类型，表示指向任意对象的指针，和 OC 中的 id 类似
 (*): 函数名
 (void *): 参数类型，void *
 4. 传递给第三个参数(函数)的`参数`
 */int result = pthread_create(&threadId, NULL, pthreadTest, cString);if (result == 0) {NSLog(@"成功");} else {NSLog(@"失败");}//*********2、NSThread*********[NSThread detachNewThreadSelector:@selector(threadTest) toTarget:self withObject:nil];//*********3、GCD*********dispatch_async(dispatch_get_global_queue(0, 0), ^{[self threadTest];});//*********4、NSOperation*********[[[NSOperationQueue alloc] init] addOperationWithBlock:^{[self threadTest];}];- (void)threadTest{NSLog(@"begin");
    NSInteger count = 1000 * 100;for (NSInteger i = 0; i < count; i++) {// 栈区
        NSInteger num = i;// 常量区
        NSString *name = @"zhang";// 堆区
        NSString *myName = [NSString stringWithFormat:@"%@ - %zd", name, num];NSLog(@"%@", myName);}NSLog(@"over");}void *pthreadTest(void *para){// 接 C 语言的字符串//    NSLog(@"===> %@ %s", [NSThread currentThread], para);// __bridge 将 C 语言的类型桥接到 OC 的类型
    NSString *name = (__bridge NSString *)(para);NSLog(@"===>%@ %@", [NSThread currentThread], name);return NULL;}
### 线程安全问题


当多个线程同时访问一块资源时，容易引发数据错乱和数据安全问题，有以下两种解决方案：


- 互斥锁
- 自旋锁


#### 互斥锁


- 用于保护临界区，确保同一时间，只有一条线程能够执行
- 如果代码中只有一个地方需要加锁，大多都使用 self，这样可以避免单独再创建一个锁对象
- 加了互斥锁的代码，当新线程访问时，如果发现其他线程正在执行锁定的代码，新线程就会进入休眠 针对互斥锁，还需要注意以下几点：
- 互斥锁的锁定范围，应该尽量小，锁定范围越大，效率越差
- 能够加锁的任意 NSObject 对象
- 锁对象一定要保证所有的线程都能够访问


#### 自旋锁


- 自旋锁与互斥锁类似，但它不是通过休眠使线程阻塞，而是在获取锁之前一直处于忙等（即原地打转，称为自旋）阻塞状态
- 使用场景：锁持有的时间短，且线程不希望在重新调度上花太多成本时，就需要使用自旋锁，属性修饰符atomic，本身就有一把自旋锁
- 加入了自旋锁，当新线程访问代码时，如果发现有其他线程正在锁定代码，新线程会用死循环的方法，一直等待锁定的代码执行完成，即不停的尝试执行代码，比较消耗性能

**【面试题】：自旋锁 vs 互斥锁**


- 同：在同一时间，保证了只有一条线程执行任务，即保证了相应同步的功能
- 不同： 互斥锁：发现其他线程执行，当前线程 休眠（即就绪状态），进入等待执行，即挂起。一直等其他线程打开之后，然后唤醒执行
- 自旋锁：发现其他线程执行，当前线程 一直询问（即一直访问），处于忙等状态，耗费的性能比较高


- 场景：根据任务复杂度区分，使用不同的锁，但判断不全时，更多是使用互斥锁去处理 当前的任务状态比较短小精悍时，用自旋锁
- 反之的，用互斥锁


#### atomic 原子锁 & nonatomic 非原子锁


atomic 和 nonatomic主要用于属性的修饰，以下是相关的一些说明：


- atomic是原子属性，是为多线程开发准备的，是默认属性！ 同一时间 单(线程)写多(线程)读的线程处理技术
- Mac开发中常用


- nonatomic 是非原子属性 没有锁！性能高！
- 移动端开发常用

atomic的底层在最早是通过轻量级自旋锁 OSSpinLock实现的，后来因为存在优先级反转问题被弃用了，现在内部已经改成使用os_unfair_lock


```objective-c
static inline void reallySetProperty(id self, SEL _cmd, id newValue, ptrdiff_t offset, bool atomic, bool copy, bool mutableCopy)
{
    if (offset == 0) {
        object_setClass(self, newValue);
        return;
    }

    id oldValue;
    id *slot = (id*) ((char*)self + offset);

    if (copy) {
        newValue = [newValue copyWithZone:nil];
    } else if (mutableCopy) {
        newValue = [newValue mutableCopyWithZone:nil];
    } else {
        if (*slot == newValue) return;
        newValue = objc_retain(newValue);
    }

    if (!atomic) {
        oldValue = *slot;
        *slot = newValue;
    } else { // 加锁修饰不加锁修饰
        spinlock_t& slotlock = PropertyLocks[slot];
        slotlock.lock();
        oldValue = *slot;
        *slot = newValue;
        slotlock.unlock();
    }

    objc_release(oldValue);
}
```


在代码中我们可以看出，

对于atomic修饰的属性，进行了spinlock_t加锁处理，但是在前文中提到OSSpinLock已经废弃了，这里的spinlock_t在底层是通过os_unfair_lock替代了OSSpinLock实现的加锁。同时为了防止哈希冲突，还是用了加盐操作


> **加盐（Salting）** 就是在密码前后加上随机字符串（称为“盐”），再进行哈希。 密码：123 盐：XyZ@9 哈希前的内容：“123XyZ@9” 哈希结果：MD5(“123XyZ@9”) = 7b3e25c0b34a7f… 同样的密码，加上不同的盐，哈希值完全不同。


```objective-c
using spinlock_t = mutex_tt<LOCKDEBUG>;class mutex_tt : nocopy_t {
    os_unfair_lock mLock;...
}
```


getter方法对atomic的处理，通setter是大致相同的。


```objective-c
id objc_getProperty(id self, SEL _cmd, ptrdiff_t offset, BOOL atomic) {
    if (offset == 0) {
        return object_getClass(self);
    }

    // Retain release world
    id *slot = (id*) ((char*)self + offset);
    if (!atomic) return *slot;

    // Atomic retain release world
    spinlock_t& slotlock = PropertyLocks[slot];
    slotlock.lock();
    id value = objc_retain(*slot);
    slotlock.unlock();

    // for performance, we (safely) issue the autorelease OUTSIDE of the spinlock.
    return objc_autoreleaseReturnValue(value);
}
```


**【面试题】atomic与nonatomic 的区别**


- nonatomic 非原子属性
- 非线程安全，适合内存小的移动设备


- atomic 原子属性(线程安全)，针对多线程设计的，默认值
- 保证同一时间只有一个线程能够写入(但是同一个时间多个线程都可以取值)
- atomic 本身就有一把锁(自旋锁) 单写多读:单个线程写入，多个线程可以读取
- 线程安全，需要消耗大量的资源 iOS 开发的建议


- 所有属性都声明为 nonatomic
- 尽量避免多线程抢夺同一块资源 尽量将加锁、资源抢夺的业务逻辑交给服务器端处理，减小移动客户端的压力 atomic修饰的属性绝对安全吗？ atomic只能保证我们的setter和getter方法的线程安全，并不能保证数据安全。


## GCD


GCD是异步执行任务的技术之一，是Apple开发的一个多核变成的较新的解决方式。它主要用于优化应用程序以支持多核处理器以及其他对称多处理系统。他是在一个线程池的基础上，并行执行任务。


### GCD核心


GCD的核心主要由 任务+队列+函数 组成


```objective-c
//********GCD基础写法********//创建任务
dispatch_block_t block = ^{NSLog(@"hello GCD");};//创建串行队列
dispatch_queue_t queue = dispatch_queue_create("com.CJL.Queue", NULL);//将任务添加到队列，并指定函数执行dispatch_async(queue, block);
#### 任务


任务就是执行操作的意思，换句话说就是我们在线程执行的那段代码，在GCD中是放在block的


- 同步执行 ( sync ) 同步添加任务到指定的队列中,在添加的任务执行结束前,会一直等待,直到里面的任务完成之后再继续执行 只能在当前线程中执行任务,不具备开启新线程的能力
- 异步执行(async) 异步添加任务到指定的队列中,它不会做任何等待,可以继续执行后面的任务 可以在新的线程中执行任务,具备开启新线程的能力 进入dispatch_sync函数后程序不会立刻返回，因此会阻塞线程 而进入dispatch_async函数后程序会立刻返回，不会妨碍进行下一步操作


#### 队列


![请添加图片描述](./csdn-154915356-ios-gcd-images/06-28545f097a134aafa4da923fc30ad0e4.png)


队列（Dispatch Queue） ： 这里的队列指执行任务的等待队列，即用来存放任务的队列。队列是一种特殊的线性表，采用FIFO（先进线出）的原则，即新任务总是被插入到队列的末尾，而读取任务的时候总是从队列的头部开始读取。每读取一个任务，从队列中释放一个任务。


![在这里插入图片描述](./csdn-154915356-ios-gcd-images/07-a16fee39e3cd484ba947d75103d0ef93.png)


![在这里插入图片描述](./csdn-154915356-ios-gcd-images/08-db2ee593d76544678761a39b510554d8.png)


- 串行队列：每次只有一个任务被执行，让一个任务接着一个的去执行
- 并行队列：可以让多个任务并发执行，可以开启多个线程，并且同时执行任务

并发队列的并发功能只有在异步函数下才可以体现出来


```objective-c
dispatch_queue_t queue = dispatch_queue_create("SerialQueue", DISPATCH_QUEUE_SERIAL);
        dispatch_queue_t queue2 = dispatch_queue_create("ConcurrentQueue", DISPATCH_QUEUE_CONCURRENT);

        //对于串行队列，GCD提供了一个特殊的队列：主队列
        dispatch_queue_t mainQueue = dispatch_get_main_queue();//获取串行队列

        //对于并发队列，GCD提供了一个全局并发队列
        dispatch_queue_t uque = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
```


![在这里插入图片描述](./csdn-154915356-ios-gcd-images/09-b886f0a3aac8429a96e6a9fbda5e625f.png)


![在这里插入图片描述](./csdn-154915356-ios-gcd-images/10-7e140e3735664994931b5c4e5f6a3a85.png)


#### 串行并行与同步异步


**同步异步函数**

同步函数（sync）和异步函数（async），只能决定能否开启新的线程执行任务，不能决定函数是串行执行还是并行执行


同步:


- 在当前线程中执行,不具备开启新线程的能力
- 队列后面的内容需要等到同步函数返回后才可以继续执行

异步：


- 拥有开创新线程的能力`,但是不一定会开启新线程
- 后面的内容不需要等异步函数返回后执行,后面的内容可以直接执行,所以需要开启新线程,因此具备开启新线程的能力 同步函数: 同步函数会阻塞当前函数的返回, 异步函数会立即执行,执行下面的代码


> 同步函数后面的内容要等到同步函数返回才可以执行,异步函数后面的内容不需要等异步函数返回才执行,可以直接执行,异步函数里面的内可能需要开启一个新线程来执行


**串行并行队列**

主要影响任务的执行方式，具体开不开开启新线程取决于上面的函数。


并行：多个任务并发执行

串行：一个任务执行完毕后才去执行下一个任务


- 如果传入的是一个并发队列，那就并发执行任务
- 如果传入一个手动创建的串行队列，那就在子队列串行执行
- 如果传入的是主队列，那就在主线程串行执行
- 如果传入的是一个主队列，那么就不会开启新线程


### GCD的基本使用


#### 同步串行


```objective-c
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        dispatch_queue_t serialQueue = dispatch_queue_create("serialQueue", DISPATCH_QUEUE_SERIAL);
        dispatch_sync(serialQueue, ^{
            for (int i = 0; i < 5; i++) {
                NSLog(@"1 %d == %@", i, [NSThread currentThread]);
            }
        });
        dispatch_sync(serialQueue, ^{
            for (int i = 0; i < 5; i++) {
                NSLog(@"2 %d == %@", i, [NSThread currentThread]);
            }
        });
    }
    return EXIT_SUCCESS;
}
```


[图片]

结论 ：同步串行队列没有开启新线程也没有异步执行


#### 异步串行


```objective-c
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        dispatch_queue_t serialQueue = dispatch_queue_create("serialQueue", DISPATCH_QUEUE_SERIAL);
        dispatch_async(serialQueue, ^{
            for (int i = 0; i < 5; i++) {
                NSLog(@"1 %d == %@", i, [NSThread currentThread]);
            }
        });
        dispatch_async(serialQueue, ^{
            for (int i = 0; i < 5; i++) {
                NSLog(@"2 %d == %@", i, [NSThread currentThread]);
            }
        });
        sleep(2);    //如果没有sleep，那么程序直接退出而没有输出
    }
    return EXIT_SUCCESS;
}
```


![在这里插入图片描述](./csdn-154915356-ios-gcd-images/11-138099d1d13642f388c618538b6bef4c.png)


**结论 ：** 同步串行队列没有开启新线程也没有异步执行


#### 异步串行


```objective-c
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        dispatch_queue_t serialQueue = dispatch_queue_create("serialQueue", DISPATCH_QUEUE_SERIAL);
        dispatch_async(serialQueue, ^{
            for (int i = 0; i < 5; i++) {
                NSLog(@"1 %d == %@", i, [NSThread currentThread]);
            }
        });
        dispatch_async(serialQueue, ^{
            for (int i = 0; i < 5; i++) {
                NSLog(@"2 %d == %@", i, [NSThread currentThread]);
            }
        });
        sleep(2);    //如果没有sleep，那么程序直接退出而没有输出
    }
    return EXIT_SUCCESS;
}
```


[图片]

这里开启了一个新线程来执行这里的代码，但是因为串行队列还是顺序执行


#### 同步并行


```objective-c
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        dispatch_queue_t serialQueue = dispatch_queue_create("serialQueue", DISPATCH_QUEUE_SERIAL);
        dispatch_queue_t concurrentQueue = dispatch_queue_create("concurrentQueue", DISPATCH_QUEUE_CONCURRENT);
        dispatch_sync(concurrentQueue, ^{
            for (int i = 0; i < 5; i++) {
                NSLog(@"1 %d == %@", i, [NSThread currentThread]);
                //sleep(1);
            }
        });
        dispatch_sync(concurrentQueue, ^{
            for (int i = 0; i < 5; i++) {
                NSLog(@"2 %d == %@", i, [NSThread currentThread]);
            }
        });
        //sleep(2);
    }
    return EXIT_SUCCESS;
}
```


[图片]

同步函数没有开辟新线程的能力，也没有体现并发的特点


#### 异步并行


```objective-c
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        dispatch_queue_t serialQueue = dispatch_queue_create("serialQueue", DISPATCH_QUEUE_SERIAL);
        dispatch_queue_t concurrentQueue = dispatch_queue_create("concurrentQueue", DISPATCH_QUEUE_CONCURRENT);
        dispatch_async(concurrentQueue, ^{
            for (int i = 0; i < 5; i++) {
                NSLog(@"1 %d == %@", i, [NSThread currentThread]);
                //sleep(1);
            }
        });
        dispatch_async(concurrentQueue, ^{
            for (int i = 0; i < 5; i++) {
                NSLog(@"2 %d == %@", i, [NSThread currentThread]);
            }
        });
        sleep(2);
    }
    return EXIT_SUCCESS;
}
```


[图片]

这里创建了新线程的同时还体现了并发执行的特点。


### 面试问题


#### 异步函数 + 并行队列


下面代码的输出顺序是什么


```objective-c
- (void)interview01{//并行队列
    dispatch_queue_t queue = dispatch_queue_create("com.CJL.Queue", DISPATCH_QUEUE_CONCURRENT);
    NSLog(@"1");// 耗时
    dispatch_async(queue, ^{
        NSLog(@"2");
        dispatch_async(queue, ^{
            NSLog(@"3");
        });
        NSLog(@"4");
    });
    NSLog(@"5");
}
    ----------打印结果-----------
输出顺序为：1 5 2 4 3
```


[图片]


主线程的任务队列为：`任务1、异步block1、任务5，`其中异步block1会比较耗费性能，任务1和任务5的任务复杂度是相同的，所以任务1和任务5优先于异步block1执行

在异步block1中，任务队列为：任务2、异步block2、任务4，其中block2相对比较耗费性能，任务2和任务4是复杂度一样，所以任务2和任务4优先于block2执行

最后执行block2中的任务3

在极端情况下，可能出现 任务2先于任务1和任务5执行，原因是出现了当前主线程卡顿或者 延迟的情况


**代码修改**

**【修改1】：** 将并行队列 改成 串行队列，对结果没有任何影响，顺序仍然是 1 5 2 4 3

**【修改2】：** 在任务5之前，休眠2s，即sleep(2)，执行的顺序为：1 2 4 3 5,原因是因为I/O的打印，相比于休眠2s，复杂度更简单，所以异步block1 会先于任务5执行。当然如果主队列堵塞，会出现其他的执行顺序


#### 异步函数嵌套同步函数 + 并发队列


![图片](./csdn-154915356-ios-gcd-images/12-bc139fac2d9d4d2d8b78a7be1c04a960.png)


任务1 和 任务5的分析同前面一致，执行顺序为 `任务1 任务5 异步block`

在异步block中，首先执行`任务2，然后走到同步block，`由于`同步函数会阻塞主线程，`所以任务4需要等待任务3执行完成后，才能执行，所以异步block中的执行顺序是：`任务2 任务3 任务4`

[图片]


#### 异步函数嵌套同步函数 + 串行队列


![在这里插入图片描述](./csdn-154915356-ios-gcd-images/13-c289bed788d04647aae2888f6915e025.png)


首先执行任务1，接下来是`异步block，并不会阻塞主线程，`相比任务5而言，复杂度更高，所以`优先执行任务5，在执行异步block`

在`异步block中，先执行任务2`，接下来是同步block，同步函数会阻塞线程，所以执行任务4需要等待任务3执行完成，而任务3的执行，需要等待异步block执行完成，相当于任务3等待任务4完成

所以就造成了`任务4等待任务3，任务3等待任务4，即互相等待的局面，就会造成死锁，这里有个`重点是`关键的堆栈 slow`


**修改** ：如果删掉任务4，执行顺序是什么？

还是会死锁，因为 **任务3等待的是异步block执行完毕，而异步block等待任务3**


### 死锁


死锁死因为资源有限以及线程的交错执行导致的。


死锁产生的四个必要条件：


- 互斥访问，在有互斥访问的情况下，线程才会出现等待
- 持有并等待，线程持有一些资源，并等待一些资源
- 资源非抢占，一旦一个资源被持有，除非持有者主动放弃，否则其他竞争者都无法获取这个资源
- 循环等待：循环等待是指存在一系列线程T0,T1,TnT0等待T1,T1等待T2,T2等待Tn这样便出现了一个循环等待

同步会阻塞当前函数的返回，异步函数会立刻返回，直接执行下面的代码

因为同步函数会阻塞当前的一个函数，等待这个函数返回后才去执行任务，因此同步函数会等待函数的返回


> 队列上是放任务，而线程是执行队列上的任务


#### 同步函数 + 主队列


![图片](./csdn-154915356-ios-gcd-images/14-2f5f90335de74a06a67051aeedd9d617.png)


主队列首先有viewDidLoad任务，而这个时候给主队列添加了一个新的任务，这个任务会等待我们的viewDidLoad执行完毕才会继续执行任务。但主队列绑定主线程，主线程正在执行，他必须空出来才能执行队列的renew，但是此时主线程被sync挂住了，他在等待主队列任务执行完才能继续。


举个例子：

你（主线程）在厨房干活的时候（执行 main 函数）

忽然接到一个命令：

“把一道菜（block）放进你的订单列表（主队列），

然后等它做完再继续干其他事。”


于是：

•        厨师（主线程）把任务（block）加进订单列表；

•        然后原地等着这道菜做完；

•        但厨房里只有他一个厨师（串行执行）；

•        可他现在在等自己；

•        所以他永远不会开始做这道菜。


#### 异步串行队列嵌套同步串行队列


![在这里插入图片描述](./csdn-154915356-ios-gcd-images/15-596da1d61be84b62a20437b229aba9fa.png)


（1）执行task1

（2）执行task5

（3）执行task2

（4）阻塞同步线程，把task3加入到队列myQueue的队尾

（5）task4需要等待task3执行完成后执行，但是此时task3又排在task4后面，所以造成了死锁


#### 信号量阻塞主线程


```objective-c
dispatch_semaphore_t sem = dispatch_semaphore_create(0);
dispatch_async(dispatch_get_main_queue(), ^{
    dispatch_semaphore_signal(sem);
    NSLog(@"the sem +1");
});
dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER);
NSLog(@"the sem -1");
```


主线程中`dispatch_semaphore_wait`一直等着`dispatch_semaphore_signal`改变信号量（+1操作），但是dispatch_semaphore_wait却阻塞了主线程导致dispatch_semaphore_signal无法执行，从而造成了死锁。


### GCD相关方法


#### dispatch_set_target_queue


dispatch_queue_create 方法生成的Queue不论是串行还是并行队列，都是和globalQueue的默认优先级相同执行优先级的线程，对于变更优先级我们需要使用`dispatch_set_target_queue函数`


```objective-c
dispatch_set_target_queue(serialQueue, globalDispatchQueueBackGround);
```


第一个是需要变更优先级的参数，第二个参数制定了与那个优先级相同的队列


![在这里插入图片描述](./csdn-154915356-ios-gcd-images/16-3a6afc702c134db3a75eb42bbf8fda5a.png)


在必须将不可并行执行的处理追加到多个Serial Dispatch Queue 中时，如果使用dispatch set target queue 西数将目标指定为某 一个Serial DispatchQueue，即可防止处理并行执行。


#### dispatch_semaphore


什么是信号量，这里引用学长的学长的一段话


> 简单起见，假设停车场只有三个车位，一开始三个车位都是空的。这时如果同时来了五辆车，看 门人允许其中三辆直接进入，然后放下车拦，剩下的车则必须在入口等待，此后来的车也都不得不在入口处等待。这时，有一辆车离开停车场，看门人得知后，打开 车拦，放入外面的一辆进去，如果又离开两辆，则又可以放入两辆，如此往复。在这个停车场系统中，车位是公共资源，每辆车好比一个线程，看门人起的就是信号量的作用


```objective-c
dispatch_semaphore_t currSingal = dispatch_semaphore_create(value);// 创建信号量,如果小于0则会返回NULL
dispatch_semaphore_signal(dispatch_semaphore_t  _Nonnull dsema); // 发送信号量让信号量加1
dispatch_semaphore_wait(dispatch_semaphore_t  _Nonnull dsema, dispatch_time_t timeout); // 可以让总信号量减1,信号量小于0的时候就会一直等待,否则就可以正常执行
```


我们要清楚需要哪一个线程等待，哪一个线程继续执行，然后使用信号量


作用 ：


- 保持线程同步，将异步任务转化为同步执行任务
- 保障线程的安全，为线程加锁


##### 并发队列里的异步任务顺序执行


```objective-c
dispatch_semaphore_t currentSingal = dispatch_semaphore_create(0);
    dispatch_queue_t que = dispatch_get_global_queue(0, 0);
    dispatch_async(que, ^{
        NSLog(@"执行任务1， %@", [NSThread currentThread]);
        dispatch_semaphore_signal(currentSingal);
    });
    dispatch_semaphore_wait(currentSingal, DISPATCH_TIME_FOREVER);

    dispatch_async(que, ^{
        NSLog(@"执行任务2， %@", [NSThread currentThread]);
        dispatch_semaphore_signal(currentSingal);
    });
    dispatch_semaphore_wait(currentSingal, DISPATCH_TIME_FOREVER);
    dispatch_async(que, ^{
        NSLog(@"执行任务3， %@", [NSThread currentThread]);
        dispatch_semaphore_signal(currentSingal);
    });
```


![图片](./csdn-154915356-ios-gcd-images/17-758b6076c7774b5abbcfc85723e590a4.png)


![图片](./csdn-154915356-ios-gcd-images/18-200ff32c2a0a439caaefbe0d3df9c17d.png)


```objective-c
/**
 * semaphore 线程同步
 */
- (void)semaphoreSync {

    NSLog(@"currentThread---%@",[NSThread currentThread]);  // 打印当前线程
    NSLog(@"semaphore---begin");

    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);

    __block int number = 0;
    dispatch_async(queue, ^{
        // 追加任务 1
        [NSThread sleepForTimeInterval:2];              // 模拟耗时操作
        NSLog(@"1---%@",[NSThread currentThread]);      // 打印当前线程

        number = 100;

        dispatch_semaphore_signal(semaphore);
    });

    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    NSLog(@"semaphore---end,number = %zd",number);
}
```


![图片](./csdn-154915356-ios-gcd-images/19-39c5125ffc224f21bb372f5b7feb1006.png)


![图片](./csdn-154915356-ios-gcd-images/20-24843211544c49279ac0297cf62ee5c1.png)


在这段代码里：

•        主线程被 wait 阻塞；

•        异步任务完成后 signal；

•        主线程恢复运行；

→ 实现了异步任务的同步等待。


##### 设置一个最大开辟的线程数 ：


在我们如果要下载很多图片的话,并发异步进行,每一个下载都会开辟一个新线程,可以我们又担心太多线程会道指内存开销太大,以及线程的上下文切换给我们的cpu带来的开销太大导致的问题所以我们要设置一下对应的一个线程最大数量就可以了


```objective-c
dispatch_semaphore_t currSingal = dispatch_semaphore_create(3);// 创建信号量,如果小于0则会返回NULL
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    for (int i = 0; i < 10; i++) {
        dispatch_async(queue, ^{
            dispatch_semaphore_wait(currSingal, DISPATCH_TIME_FOREVER);
            NSLog(@"执行任务, %d", i);
            sleep(1);
            NSLog(@"完成任务, %d", i);
            dispatch_semaphore_signal(currSingal);
        });
    }
```


![图片](./csdn-154915356-ios-gcd-images/21-e91ac3b30c62447c84627ec13e88ce43.png)


在诶里可以看出我们这里最开始的三个异步操作,剩下的需要同步等待,只有释放出来的操作才可以进行异步操作

这里其实出现了一个优先级反转的问题,这里我们看一下:如果打印我们的NSThread,他会出现什么:


![图片](./csdn-154915356-ios-gcd-images/22-06f7260a168a4660919b96dd58e953a0.png)


这个报错锁我们的User-initiated这个线程在等待交底优先级的线程.出现了一个优先级反转的内容:


_**优先级反转**_

假设现在有3个任务A、B｀C’它们的优先 级为A＞B＞C任务C在运行时持有一把锁’然后它被高优先级的任务A抢占了（任 务C的锁没有被释放）。此时任务A恰巧也想申请任务C持有的锁’但是申请失败,因 此进入阻塞状态等待任务C放锁。此时’任务B、C都处于可以运行的状态,由于任务 B的优先级高于C,因此B优先运行°综合观察该情况’就会发现任务B好像优先级高 于任务A’先于任务A执行。


![图片](./csdn-154915356-ios-gcd-images/23-f68a75ac91cd47ae9bee6c4cb2136089.png)


这里我们来理解一下上面为什么会出现一个优先级反转的问题：


- 前三个任务线进入工作状态
- 后面4 - 10个任务被阻塞
- 系统会在这些阻塞的线程中，挑选一个来唤醒（注意这里并不一定考虑线程的一个优先级）
- 这时候如果某一个优先级比较低的线程被唤醒会开始执行
- 此时另一个高优先级的线程也在等待信号量,去不许等这个低优先级的线程先完成,这样那个自高优先级的线程就被低优先级的线程卡住了,这就是我们这里说的优先级反转的问题


> 信号量调度是公平队列（FIFO / 不考虑QoS）+ 系统经常调度是根据优先级来的，两者机制不一致，所以可能造成高优先级线程因为信号量资源被低优先级线程占用而“被阻塞”


所以在iOS中为了避免我们出现优先级反转的问题,尽量减少采用信号量的方式


##### Dispatch Semaphore 线程安全和线程同步（为线程加锁）


在程序执行时多个任务可能会对同—份数据产生 竞争’因此任务会使用锁来保护共享数据°

线程安全：如果你的代码所在的进程中有多个线程在同时运行，而这些线程可能会同时运行这段代码。如果每次运行结果和单线程运行的结果是一样的，而且其他的变量的值也和预期的是一样的，就是线程安全的。


若每个线程中对全局变量、静态变量只有读操作，而无写操作，一般来说，这个全局变量是线程安全的；若有多个线程同时执行写操作（更改变量），一般都需要考虑线程同步，否则的话就可能影响线程安全。


线程同步：可理解为线程 A 和 线程 B 一块配合，A 执行到一定程度时要依靠线程 B 的某个结果，于是停下来，示意 B 运行；B 依言执行，再将结果给 A；A 再继续操作。


这里就涉及到之前了解过的售票问题，下面，我们模拟火车票售卖的方式，实现 NSThread 线程安全和解决线程同步问题。


```objective-c
/**
 * 售卖火车票（线程安全）
 */
- (void)saleTicketSafe {
    while (1) {
        // 相当于加锁
        dispatch_semaphore_t semaphoreLock = dispatch_semaphore_create(1);        dispatch_semaphore_wait(semaphoreLock, DISPATCH_TIME_FOREVER);

        if (self.ticketSurplusCount > 0) {  // 如果还有票，继续售卖
            self.ticketSurplusCount--;
            NSLog(@"%@", [NSString stringWithFormat:@"剩余票数：%ld 窗口：%@", (long)self.ticketSurplusCount, [NSThread currentThread]]);
            [NSThread sleepForTimeInterval:0.2];
        } else { // 如果已卖完，关闭售票窗口
            NSLog(@"所有火车票均已售完");

            // 相当于解锁
            dispatch_semaphore_signal(semaphoreLock);
            break;
        }

        // 相当于解锁
        dispatch_semaphore_signal(semaphoreLock);
    }
}
```


这就保证了我们的线程一次只能处理一个任务，也就使我们的进程由并发队列+异步任务转换为了并发队列+同步任务


#### dispatch_after


表示在某队列中的block延迟执行

应用：在主队列延迟执行一项任务，如viewDidLoad后延迟1s，提示一个alertView（是延迟加入到队列，而不是延迟执行）


```objective-c
dispatch_once
- (void)cjl_testOnce{
    /*
     dispatch_once保证在App运行期间，block中的代码只执行一次
     应用场景：单例、method-Swizzling
     */
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        //创建单例、method swizzled或其他任务
        NSLog(@"创建单例");
    });
}
#### dispatch_apply


按照指定的次数将指定的任务追加到指定的队列中，快速迭代一组任务，并等待全部队列执行结束。主要用于需要快速并行处理大量相同类型任务的场景，如图像处理、数据处理或任何形式的批处理任务。


在串行队列中无法体现出它的作用，因为队列中的任务始终是一次执行一个。


在并发队列中如果开启了多个线程的话就能在多个线程中同时展开执行，效率特别高。


```objective-c
- (void)cjl_testApply{
    /*
     dispatch_apply将指定的Block追加到指定的队列中重复执行，并等到全部的处理执行结束——相当于线程安全的for循环

     应用场景：用来拉取网络数据后提前算出各个控件的大小，防止绘制时计算，提高表单滑动流畅性
     - 添加到串行队列中——按序执行
     - 添加到主队列中——死锁
     - 添加到并发队列中——乱序执行
     - 添加到全局队列中——乱序执行
     */

    dispatch_queue_t queue = dispatch_queue_create("CJL", DISPATCH_QUEUE_SERIAL);
    NSLog(@"dispatch_apply前");
    /**
         param1：重复次数
         param2：追加的队列
         param3：执行任务
         */
    dispatch_apply(10, queue, ^(size_t index) {
        NSLog(@"dispatch_apply 的线程 %zu - %@", index, [NSThread currentThread]);
    });
    NSLog(@"dispatch_apply后");
}
#### dispatch_group_t


notify的作用就是：**当 group 中所有任务完成（enter/leave 配对完或者 dispatch_group_async 内 block 执行完）时，执行 block。**


##### dispatch_group_async + dispatch_group_notify


```objective-c
- (void)cjl_testGroup1{
    /*
     dispatch_group_t：调度组将任务分组执行，能监听任务组完成，并设置等待时间

     应用场景：多个接口请求之后刷新页面
     */

    dispatch_group_t group = dispatch_group_create();
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);

    dispatch_group_async(group, queue, ^{
        NSLog(@"请求一完成");
    });

    dispatch_group_async(group, queue, ^{
        NSLog(@"请求二完成");
    });

    // 当group内所有任务完成时，这个block会在主线程执行。很适合UI刷新，不阻塞主线程

    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        NSLog(@"刷新页面");
    });
}
##### dispatch_group_enter + dispatch_group_leave + dispatch_group_notify


```objective-c
- (void)cjl_testGroup2{
    /*
     dispatch_group_enter和dispatch_group_leave成对出现，使进出组的逻辑更加清晰
     */
    dispatch_group_t group = dispatch_group_create();
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);

    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        NSLog(@"请求一完成");
        dispatch_group_leave(group);
    });

    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        NSLog(@"请求二完成");
        dispatch_group_leave(group);
    });

    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        NSLog(@"刷新界面");
    });
}
```


- dispatch_group_enter(group)：标记“组里有一个任务开始了”。
- dispatch_group_leave(group)：标记“组里的一个任务完成了”。
- **enter/leave 必须配对出现，**否则组永远不会完成。


> 可以理解为给 group 维护了一个计数器：每次 enter +1，每次 leave -1，当计数器归零时，group 完成。


##### dispatch_wait


![图片](./csdn-154915356-ios-gcd-images/24-8d8beb2d29034599859f3a7f76938392.png)


wait会阻塞线程，notify不会阻塞 是异步等待


```objective-c
- (void)cjl_testGroup3{
    /*
     long dispatch_group_wait(dispatch_group_t group, dispatch_time_t timeout)

     group：需要等待的调度组
     timeout：等待的超时时间（即等多久）
        - 设置为DISPATCH_TIME_NOW意味着不等待直接判定调度组是否执行完毕
        - 设置为DISPATCH_TIME_FOREVER则会阻塞当前调度组，直到调度组执行完毕


     返回值：为long类型
        - 返回值为0——在指定时间内调度组完成了任务
        - 返回值不为0——在指定时间内调度组没有按时完成任务

     */
    dispatch_group_t group = dispatch_group_create();
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);

    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        NSLog(@"请求一完成");
        dispatch_group_leave(group);
    });

    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        NSLog(@"请求二完成");
        dispatch_group_leave(group);
    });

//    long timeout = dispatch_group_wait(group, DISPATCH_TIME_NOW);
//    long timeout = dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
    long timeout = dispatch_group_wait(group, dispatch_time(DISPATCH_TIME_NOW, 1 *NSEC_PER_SEC));
    NSLog(@"timeout = %ld", timeout);
    if (timeout == 0) {
        NSLog(@"按时完成任务");
    }else{
        NSLog(@"超时");
    }

    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        NSLog(@"刷新界面");
    });
}
#### dispatch_barrier_sync & dispatch_barrier_async


dispatch_barrier_async 是 GCD（Grand Central Dispatch）中的一个方法，它用于在并发队列中插入一个“栅栏”（barrier）。栅栏的作用是在其前面的任务执行完毕后，再执行栅栏任务，然后再执行栅栏之后的任务。这可以用于确保在多线程环境中某些操作的顺序性和互斥性。

栅栏函数，主要有两种使用场景：串行队列、并发队列


```objective-c
- (void)cjl_testBarrier{
    /*
     dispatch_barrier_sync & dispatch_barrier_async

     应用场景：同步锁

     等栅栏前追加到队列中的任务执行完毕后，再将栅栏后的任务追加到队列中。
     简而言之，就是先执行栅栏前任务，再执行栅栏任务，最后执行栅栏后任务

     - dispatch_barrier_async：前面的任务执行完毕才会来到这里
     - dispatch_barrier_sync：作用相同，但是这个会堵塞线程，影响后面的任务执行

     - dispatch_barrier_async可以控制队列中任务的执行顺序，
     - 而dispatch_barrier_sync不仅阻塞了队列的执行，也阻塞了线程的执行（尽量少用）
     */

    [self cjl_testBarrier1];
    [self cjl_testBarrier2];
}
- (void)cjl_testBarrier1{
    //串行队列使用栅栏函数

    dispatch_queue_t queue = dispatch_queue_create("CJL", DISPATCH_QUEUE_SERIAL);

    NSLog(@"开始 - %@", [NSThread currentThread]);
    dispatch_async(queue, ^{
        sleep(2);
        NSLog(@"延迟2s的任务1 - %@", [NSThread currentThread]);
    });
    NSLog(@"第一次结束 - %@", [NSThread currentThread]);

    //栅栏函数的作用是将队列中的任务进行分组，所以我们只要关注任务1、任务2
    dispatch_barrier_async(queue, ^{
        NSLog(@"------------栅栏任务------------%@", [NSThread currentThread]);
    });
    NSLog(@"栅栏结束 - %@", [NSThread currentThread]);

    dispatch_async(queue, ^{
        sleep(2);
        NSLog(@"延迟2s的任务2 - %@", [NSThread currentThread]);
    });
    NSLog(@"第二次结束 - %@", [NSThread currentThread]);
}
- (void)cjl_testBarrier2{
    //并发队列使用栅栏函数

    dispatch_queue_t queue = dispatch_queue_create("CJL", DISPATCH_QUEUE_CONCURRENT);

    NSLog(@"开始 - %@", [NSThread currentThread]);
    dispatch_async(queue, ^{
        sleep(2);
        NSLog(@"延迟2s的任务1 - %@", [NSThread currentThread]);
    });
    NSLog(@"第一次结束 - %@", [NSThread currentThread]);

    //由于并发队列异步执行任务是乱序执行完毕的，所以使用栅栏函数可以很好的控制队列内任务执行的顺序
    dispatch_barrier_async(queue, ^{
        NSLog(@"------------栅栏任务------------%@", [NSThread currentThread]);
    });
    NSLog(@"栅栏结束 - %@", [NSThread currentThread]);

    dispatch_async(queue, ^{
        sleep(2);
        NSLog(@"延迟2s的任务2 - %@", [NSThread currentThread]);
    });
    NSLog(@"第二次结束 - %@", [NSThread currentThread]);
}
```


栅栏函数最典型的用途就是实现读写安全：


```objective-c
dispatch_queue_t rwQueue = dispatch_queue_create("com.example.rwQueue", DISPATCH_QUEUE_CONCURRENT);
NSMutableArray *array = [NSMutableArray array];

// 写操作：用 barrier 独占执行
dispatch_barrier_async(rwQueue, ^{
    [array addObject:@"数据"];
    NSLog(@"写入完成");
});

// 读操作：可以并发执行
dispatch_async(rwQueue, ^{
    NSLog(@"读取数据1：%@", array);
});
dispatch_async(rwQueue, ^{
    NSLog(@"读取数据2：%@", array);
});
```


读操作可以并发，提高性能。写操作被barrier独占，保证数据安全


#### dispatch_source_t


暂时无法在飞书文档外展示此内容


GCD 中**最底层、最强大的系统事件处理机制之一**，被称为 **“事件源 (Dispatch Source)”。**

它可以用来监听 系统底层事件（如文件变化、计时器、进程、信号、网络、I/O 等）。

主要用于计时操作，其原因是因为他创建的timer不依赖于RunLoop，且即使精准度比NSTimer高


```objective-c
- (void)cjl_testSource{
    /*
     dispatch_source

     应用场景：GCDTimer
     在iOS开发中一般使用NSTimer来处理定时逻辑，但NSTimer是依赖Runloop的，而Runloop可以运行在不同的模式下。如果NSTimer添加在一种模式下，当Runloop运行在其他模式下的时候，定时器就挂机了；又如果Runloop在阻塞状态，NSTimer触发时间就会推迟到下一个Runloop周期。因此NSTimer在计时上会有误差，并不是特别精确，而GCD定时器不依赖Runloop，计时精度要高很多

     dispatch_source是一种基本的数据类型，可以用来监听一些底层的系统事件
        - Timer Dispatch Source：定时器事件源，用来生成周期性的通知或回调
        - Signal Dispatch Source：监听信号事件源，当有UNIX信号发生时会通知
        - Descriptor Dispatch Source：监听文件或socket事件源，当文件或socket数据发生变化时会通知
        - Process Dispatch Source：监听进程事件源，与进程相关的事件通知
        - Mach port Dispatch Source：监听Mach端口事件源
        - Custom Dispatch Source：监听自定义事件源

     主要使用的API：
        - dispatch_source_create: 创建事件源
        - dispatch_source_set_event_handler: 设置数据源回调
        - dispatch_source_merge_data: 设置事件源数据
        - dispatch_source_get_data： 获取事件源数据
        - dispatch_resume: 继续
        - dispatch_suspend: 挂起
        - dispatch_cancle: 取消
     */

    //1.创建队列
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    //2.创建timer
    dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
    //3.设置timer首次执行时间，间隔，精确度
    dispatch_source_set_timer(timer, DISPATCH_TIME_NOW, 2.0*NSEC_PER_SEC, 0.1*NSEC_PER_SEC);
    //4.设置timer事件回调
    dispatch_source_set_event_handler(timer, ^{
        NSLog(@"GCDTimer");
    });
    //5.默认是挂起状态，需要手动激活
    dispatch_resume(timer);

}
#### disoatch_suspend & dispatch_resume


这两个函数可以随时挂起某个队列，等需要执行的时候恢复即可


```objective-c
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_suspend(queue);
    dispatch_resume(queue);
```


## GCD任务能取消吗？


GCD本身没有提供取消任务的API，但是可以通过操作来间接设置取消任务比如：


在 block 内部添加一个布尔变量作为取消标志，然后在外部改变这个标志的值。block 在执行时可以检查这个标志是否被设置为取消状态，如果被设置，则提前结束 block 的执行。


DispatchSource 可以用来监控文件描述符、定时器等事件，但也可以用于模拟任务取消。可以创建一个 DispatchSourceTimer，并在定时器触发时检查是否应该取消任务。


## 如何使用GCD实现一个常驻线程？


- 使用无限循环 在一个线程中使用无限循环（如 while 循环）来持续执行任务或监听事件。为了防止线程占用过多的 CPU 资源，可以在循环中加入适当的延时。
- 使用 dispatch_source_t dispatch_source_t 是 GCD 中一种可以用于监听事件的对象，比如定时器、文件描述符等。可以创建一个 dispatch_source_t 定时器，让它定期触发并执行任务。
- 使用NSThread 使用NSThread创建线程并在其中执行无限循环或监听事件。


## 怎样利用GCD实现多读单写呢（怎么实现一个多读单写的模型）？


通过栅栏异步调用的方式，分配写操作到并发队列当中。

---

原文发布于 CSDN：[【iOS】多线程与GCD](https://blog.csdn.net/2402_86720949/article/details/154915356)
