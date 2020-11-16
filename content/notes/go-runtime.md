+++
title = "Go Runtime笔记"
date = 2019-12-03T00:00:00+08:00
tags = ["go"]
categories = [""]
draft = false
+++

Go Runtime是go语言运行的基础设施，Runtime的功能：

1. 协程的创建和调度
2. 内存分配和回收管理
3. map，channel，mutex的创建和实现
4. pprof和race等检测的实现

## 协程调度

## Go的并发模型

- 基于CSP（Communicating Sequential Processes）
- 不要用共享内存来通信，用通信来共享内存（don't communicate by sharing memory share memory by communicating）

### Goroutine

- 并发执行的最小实体
- 初始栈大小为2KB，最多增长到1GB
- 保存和恢复执行流的上下文

### 线程的实现模型

- 用户级线程模型：N个用户线程对应一个内核线程，由用户自己来进行线程调度，不需要上下文切换，线程会阻塞同进程中的其他线程。
- 内核级线程模型：一个用户线程对应一个内核线程，由操作系统来进行线程调度，多个线程可以并行运行，需要上下文切换。
- 两级线程模型：N个用户线程对应M个内核线程，用户线程跟内核线程之间实现动态关联，Go的Runtime属于这种模式。

### G-M模型

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/693f6dd1-861e-41d3-a3b5-83f82d877b89/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/1.png)

Go在最初的1.0版本时Goroutine的调度模型为G-M模型：

- 创建的Goroutine放入一个全局队列中
- 每个M与一个Core绑定，通过findG()从队列中获取Goroutine来调度

问题：

- 全局锁单一，创建、调度Goroutine都要给队列上锁，造成性能下降
- M之间经常要传递可运行的Goroutine，造成调度延迟增大
- 每个M都有memory cache，造成额外内存消耗

### G-P-M模型

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/42d02189-cc74-4916-bfb9-b759c973bc3e/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/2.png)

Go在1.1版本后将Goroutinue版本改为了G-P-M模型：

- G：需要调度的Goroutine(用户代码)
- P：逻辑处理器，与M绑定，提供执行环境，内存分配和任务队列
- M：工作线程和执行者，与内核线程绑定

这个模型实现了*Work Stealing*算法，解决了之前调度存在的问题：

- 每个P维护一个G队列，新的G放到local runq中，满了再放到全局队列
- 如果存在P中的队列为空时，M会随机挑选一个P，将其中的G放到自己队列中
- memory cache与P绑定，M进行系统调用时与P解绑，确保缓存被正在运行Go代码的M利用
- 当G进行网络/锁切换时，G会放到等待队列中，M与G解绑，重新调度运行新的G
- 当G阻塞在系统调用时，P与阻塞在系统调用的M解绑，寻找其他的idle M，系统调用结束后G重新寻找一个idle P，放入其runq中

### Goroutine基本状态

- **_Gidle**：Goroutine被分配内存，但还没初始化
- **_Grunnable**：Goroutine已经放到了runq里，但还没被运行
- **_Grunning**：Goroutine与M和P绑定，正在执行用户代码
- **_Gsyscall**：Goroutine正在执行系统调用，此时G与M处于绑定状态
- **_Gwaiting**：Goroutine因阻塞(IO， channel，锁)处于等待并可被调度状态中

### 协程抢占 - sysmon

sysmon是一个特殊的轻量级协程，sysmon对于运行过久的G设置抢占标识，对于过久系统调用的P，进行M和P的分离，防止P被占用过久影响调度。

sysmon调用retake()时把P的gp.stackguard0设置为stackPreempt，导致P中的执行的G在下次函数调用时触发morestack()，morestack()除了检查是否需要扩张栈，同时还检查是否当前协程需要抢占。

## 内存分配

## Go的内存分配

- 基于 `TCMalloc` 算法
- 内存管理基本单元是mspan ，每个span有若干个页来组成，每个mspan用于一个范围内的内存分配需求
- 极小对象分配在一个object里，使用tiny分配器分配内存，一般对象用mspan分配器分配内存，大对象由mheap分配内存
- 优先从当前P的mcache分配，没有的话去mcentral查找，没有的话再去全局的mheap查找
- Go对于GC后回收的内存页, 并不是马上归还给操作系统, 而是会延迟归还, 用于满足未来的内存需求

Go的内存分配器在分配对象时，根据对象的大小，分成三类：小对象（小于等于16B）、一般对象（大于16B，小于等于32KB）、大对象（大于32KB）

大体上的分配流程：

- > 32KB 的对象，直接从mheap上分配

- <=16B 的对象使用mcache的tiny分配器分配

- (16B,32KB] 的对象，首先计算对象的规格大小，然后使用mcache中相应规格大小的mspan分配

- 如果mcache没有相应规格大小的mspan，则向mcentral申请

- 如果mcentral没有相应规格大小的mspan，则向mheap申请

- 如果mheap中也没有合适大小的mspan，则向操作系统申请

## 垃圾回收

Golang的混合写屏障结合了**插入屏障(插入屏障拦截将白色指针插入黑色对象的操作，标记其对应对象为灰色状态)、删除屏障（保护灰色对象到白色对象的路径不会断）**，防止对象被漏标记。

## Golang GC

- Go在1.3之前用的是Mark-Sweep算法
- Go在1.3版本把Sweep改成了并行操作
- Go在1.5后改为采用三色标记法，并发标记和清理，混合写屏障，效率有重大提升

## 三色标记

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/2757e142-5cfc-4ae3-b16d-6d146b01ddbf/gc-1.gif](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/3.gif)

1. 初始化三个集合：白、灰、黑
2. 将所有对象放入白色集合
3. 从根节点开始遍历扫描所有对象，把遍历的对象从白色集合放入灰色集合
4. 遍历灰色集合，将灰色对象引用的白色对象从白色集合放入灰色集合，再把自身从灰色集合放入黑色集合
5. 重复 4 直到灰色中无任何对象，所有可达对象都被标记
6. 通过write-barrier检测对象有变化，重复以上操作
7. 收集所有白色对象

## 写屏障

三色标记需要维护不变性条件：黑色对象不能引用无法被灰色对象可达的白色对象。