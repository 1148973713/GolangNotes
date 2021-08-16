**线程（Thread）**：有时被称为轻量级进程(Lightweight Process，LWP），是程序执行流的最小单元。线程是进程中的一个实体，是被系统独立调度和分派的基本单位，线程自己不拥有系统资源，只拥有一点儿在运行中必不可少的资源，但它可与同属一个进程的其它线程共享进程所拥有的全部资源。

**协程（coroutine）**：又称微线程与子例程（或者称为函数）一样，协程（coroutine）也是一种程序组件。相对子例程而言，协程更为一般和灵活，但在实践中使用没有子例程那样广泛。和线程类似，共享堆，不共享栈，协程的切换一般由程序员在代码中显式控制。它避免了上下文切换的额外耗费，兼顾了多线程的优点，简化了高并发程序的复杂。

**一、Go并发模型**

Go的CSP并发模型，是通过`goroutine`和`channel`来实现的。

- `goroutine` 是Go语言中并发的执行单位。有点抽象，其实就是和传统概念上的”线程“类似，可以理解为”线程“。
- `channel`是Go语言中各个并发结构体(`goroutine`)之前的通信机制。 通俗的讲，就是各个`goroutine`之间通信的”管道“，有点类似于Linux中的管道。

**二、Go并发模型实现原型**

操作系统根据资源访问权限的不同，体系架构可分为**用户空间和内核空间**；内核空间主要操作访问CPU资源、I/O资源、内存资源等硬件资源，为上层应用程序提供最基本的基础资源，用户空间呢就是上层应用程序的固定活动空间，用户空间不可以直接访问资源，必须通过“系统调用”、“库函数”或“Shell脚本”来调用内核空间提供的资源。

### 用户级线程模型

![img](https:////upload-images.jianshu.io/upload_images/1190574-21814076326bf301.png?imageMogr2/auto-orient/strip|imageView2/2/w/594/format/webp)

如图所示，多个用户态的线程对应着一个内核线程，程序线程的创建、终止、切换或者同步等线程工作必须自身来完成。它可以做快速的上下文切换。缺点是不能有效利用多核CPU。

### 内核级线程模型

![img](https:////upload-images.jianshu.io/upload_images/1190574-72bb5dd79ccc3bde.png?imageMogr2/auto-orient/strip|imageView2/2/w/394/format/webp)

这种模型直接调用操作系统的内核线程，所有线程的创建、终止、切换、同步等操作，都由内核来完成。一个用户态的线程对应一个系统线程，它可以利用多核机制，但上下文切换需要消耗额外的资源。C++就是这种。

### 两级线程模型

![img](https:////upload-images.jianshu.io/upload_images/1190574-4a9b12e466b4e0d5.png?imageMogr2/auto-orient/strip|imageView2/2/w/404/format/webp)

这种模型是介于用户级线程模型和内核级线程模型之间的一种线程模型。这种模型的实现非常复杂，和内核级线程模型类似，一个进程中可以对应多个内核级线程，但是进程中的线程不和内核线程一一对应；这种线程模型会先创建多个内核级线程，然后用自身的用户级线程去对应创建的多个内核级线程，自身的用户级线程需要本身程序去调度，内核级的线程交给操作系统内核去调度。

M个用户线程对应N个系统线程，缺点增加了调度器的实现难度。

Go语言的线程模型就是一种特殊的两级线程模型（GPM调度模型）。

## Go线程实现模型MPG

`M`指的是`Machine`，一个`M`直接关联了一个内核线程。由操作系统管理。
 `P`指的是”processor”，代表了`M`所需的上下文环境，也是处理用户级代码逻辑的处理器。它负责衔接M和G的调度上下文，将等待执行的G与M对接。
 `G`指的是`Goroutine`，其实本质上也是一种轻量级的线程。包括了调用栈，重要的调度信息，例如channel等。

P的数量由环境变量中的`GOMAXPROCS`决定，通常来说它是和核心数对应，例如在4Core的服务器上回启动4个线程。G会有很多个，每个P会将Goroutine从一个就绪的队列中做Pop操作，为了减小锁的竞争，通常情况下每个P会负责一个队列。

三者关系如下图所示：

![img](https:////upload-images.jianshu.io/upload_images/1190574-6bff3fbee55dc676.png?imageMogr2/auto-orient/strip|imageView2/2/w/400/format/webp)

以上这个图讲的是两个线程(内核线程)的情况。一个**M**会对应一个内核线程，一个**M**也会连接一个上下文**P**，一个上下文**P**相当于一个“处理器”，一个上下文连接一个或者多个Goroutine。为了运行goroutine，线程必须保存上下文。

上下文P(Processor)的数量在启动时设置为`GOMAXPROCS`环境变量的值或通过运行时函数`GOMAXPROCS()`。通常情况下，在程序执行期间不会更改。上下文数量固定意味着只有固定数量的线程在任何时候运行Go代码。我们可以使用它来调整Go进程到个人计算机的调用，例如4核PC在4个线程上运行Go代码。

图中P正在执行的`Goroutine`为蓝色的；处于待执行状态的`Goroutine`为灰色的，灰色的`Goroutine`形成了一个队列`runqueues`。

Go语言里，启动一个goroutine很容易：go function 就行，所以每有一个go语句被执行，runqueue队列就在其末尾加入一个goroutine，一旦上下文运行goroutine直到调度点，它会从其runqueue中弹出goroutine，设置堆栈和指令指针并开始运行goroutine。

![img](https:////upload-images.jianshu.io/upload_images/1190574-b58e548fb3c0b069.png?imageMogr2/auto-orient/strip|imageView2/2/w/800/format/webp)

#### 抛弃P(Processor)

你可能会想，为什么一定需要一个上下文，我们能不能直接除去上下文，让`Goroutine`的`runqueues`挂到M上呢？答案是不行，需要上下文的目的，是让我们可以直接放开其他线程，当遇到内核线程阻塞的时候。

一个很简单的例子就是系统调用`sysall`，一个线程肯定不能同时执行代码和系统调用被阻塞，这个时候，此线程M需要放弃当前的上下文环境P，以便可以让其他的`Goroutine`被调度执行。

![img](https:////upload-images.jianshu.io/upload_images/1190574-2746ae1e49834c80.png?imageMogr2/auto-orient/strip|imageView2/2/w/550/format/webp)

如上图左图所示，M0中的G0执行了syscall，然后就创建了一个M1(也有可能来自线程缓存)，（转向右图）然后M0丢弃了P，等待syscall的返回值，M1接受了P，将·继续执行`Goroutine`队列中的其他`Goroutine`。

当系统调用syscall结束后，M0会“偷”一个上下文，如果不成功，M0就把它的Gouroutine G0放到一个全局的runqueue中，将自己置于线程缓存中并进入休眠状态。全局runqueue是各个P在运行完自己的本地的Goroutine runqueue后用来拉取新goroutine的地方。P也会周期性的检查这个全局runqueue上的goroutine，否则，全局runqueue上的goroutines可能得不到执行而饿死。

#### 均衡的分配工作

按照以上的说法，上下文P会定期的检查全局的goroutine 队列中的goroutine，以便自己在消费掉自身Goroutine队列的时候有事可做。假如全局goroutine队列中的goroutine也没了呢？就从其他运行的中的P的runqueue里偷。

每个P中的`Goroutine`不同导致他们运行的效率和时间也不同，在一个有很多P和M的环境中，不能让一个P跑完自身的`Goroutine`就没事可做了，因为或许其他的P有很长的`goroutine`队列要跑，得需要均衡。
 该如何解决呢？

Go的做法倒也直接，从其他P中偷一半！

![img](https:////upload-images.jianshu.io/upload_images/1190574-d0442131f9b0a505.png?imageMogr2/auto-orient/strip|imageView2/2/w/550/format/webp)