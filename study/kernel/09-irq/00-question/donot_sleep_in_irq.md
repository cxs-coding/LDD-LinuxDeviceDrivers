---

title: 使用 Hexo 搭建 GitHub Page 博客(一)
date: 2018-09-02 18:40
author: gatieme
tags: hexo
categories:
        - hexo
thumbnail:
blogexcerpt: 博文摘要

---

| CSDN | GitHub | Hexo |
|:----:|:------:|:----:|
| [Aderstep--紫夜阑珊-青伶巷草](http://blog.csdn.net/gatieme) | [`AderXCoding/system/tools`](https://github.com/gatieme/AderXCoding/tree/master/system/tools) | [gatieme.github.io](https://gatieme.github.io) |

<br>

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议</a>进行许可, 转载请注明出处, 谢谢合作

因本人技术水平和知识面有限, 内容如有纰漏或者需要修正的地方, 欢迎大家指正, 也欢迎大家提供一些其他好的调试工具以供收录, 鄙人在此谢谢啦

<br>


#1	疑问
-------


这个问题实际上是一个老生常谈的问题.

其中一个比较流行的答案就是 :

>中断没有进程上下文, 而所有的进程调度都是以进程为基础的. 如果睡眠之后, 进程调度器没法来唤醒它. 
>
>仔细想想，这个答案其实不然.
>
>在没有实现或配置内核栈中断栈分离的时候, 中断发生时会借用当前被中断进程的 Kernel stack, 所以实际上中断是借宿在这个进程上, 这个时候中断睡眠是完全可以的或者说是可以实现的.(此处我们排除这个需求的不合理性), 中断上下文会保存在这个进程的 stack 上, 等到这个进程被唤醒时, 会从中断ISR中继续执行.
>
>如果内核栈和中断栈分离, 那么两者是无关的. 中断中没有进程上下文, 那么如果进行睡眠, 内核也并不是无法继续处理, 依旧可以选择一个合适的进程(比如被中断的进程)过来, 然后等中断睡眠醒来, 再继续处理中断. 理论上也可以实现.
>
>所以, 中断没有进程上下文 不是此问题的根本原因.



#2	分析
-------


##2.1	中断中睡眠有意义么(合理性)
-------

首先中断是什么?

> 中断是指计算机运行过程中, 出现某些意外情况需主机干预时, 机器能自动停止正在运行的程序并转入处理新情况的程序, 处理完毕后又返回原被暂停的程序继续运行.
>
>指处理机处理程序运行中出现的紧急事件的整个过程. 程序运行过程中, 系统外部、系统内部或者现行程序本身若出现紧急事件, 处理机立即中止现行程序的运行. 自动转入相应的处理程序(中断服务程序), 待处理完后, 再返回原来的程序运行.


这里传递了几个意思:

1.	中断的优先级很高, 必须要立即处理, 比任何进程的优先级都要高, 需要暂停程序的运行而去立马处理.

2.	中断不应该依赖于任何进程.

既然是这么重要的事情, 那么所以你为什么要问 "Linux 为什么中断不允许睡眠" 这样的问题.

很明显

从来没有人想过, 要在中断中支持睡眠这样的操作, 因为 "它" 完全不合理.

既然不合理, 那么 Linux 必然也是按照 "不允许睡眠" 这么实现的. 


##2.2	Linux 中断就是这么设计的(实现)
-------


先把中断处理流程给出来

 


```cpp
1. 进入中断处理程序
2. 保存关键上下文
3. 开中断（sti指令）
4. 进入中断处理程序的 handler
5. 关中断（cli指令）
6. 写EOI寄存器（表示中断处理完成）
7. 开中断
```

硬中断, 对应于上图的1、2、3步骤, 在这几个步骤中, 所有中断是被屏蔽的, 如果在这个时候睡眠了, 操作系统不会收到任何中断(包括时钟中断), 系统就基本处于瘫痪状态(例如调度器依赖的时钟节拍没有等等…)

软中断, 对应上图的4（当然，准确的说应该是4步骤的后面一点，先把话说保险点，免得思一克又开始较真 ）。这个时候不能睡眠的关键是因为上下文。
大家知道操作系统以进程调度为单位，进程的运行在进程的上下文中，以进程描述符作为管理的数据结构。进程可以睡眠的原因是操作系统可以切换不同进程的上下文，进行调度操作，这些操作都以进程描述符为支持。
中断运行在中断上下文，没有一个所谓的中断描述符来描述它，它不是操作系统调度的单位。一旦在中断上下文中睡眠，首先无法切换上下文（因为没有中断描述符，当前上下文的状态得不到保存），其次，没有人来唤醒它，因为它不是操作系统的调度单位.

此外，中断的发生是非常非常频繁的，在一个中断睡眠期间，其它中断发生并睡眠了，那很容易就造成中断栈溢出导致系统崩溃。
如 果上述条件满足了（也就是有中断描述符，并成为调度器的调度单位，栈也不溢出了，理论上是可以做到中断睡眠的），中断是可以睡眠的，但会引起很多问题.例 如，你在时钟中断中睡眠了，那操作系统的时钟就乱了，调度器也了失去依据；例如，你在一个IPI（处理器间中断）中，其它CPU都在死循环等你答复，你确 睡眠了，那其它处理器也不工作了；例如，你在一个DMA中断中睡眠了，上面的进程还在同步的等待I/O的完成，性能就大大降低了……还可以举出很多例子。 所以，中断是一种紧急事务，需要操作系统立即处理，不是不能做到睡眠，是它没有理由睡眠。

======================================================


1.	中断处理的时候,不应该发生进程切换，因为在中断 context 中，唯一能打断当前中断handler的只有更高优先级的中断，它不会被进程打断，如果在 中断context中休眠，则没有办法唤醒它，因为所有的wake_up_xxx都是针对某个进程而言的，而在中断context中，没有进程的概念，没 有一个task_struct（这点对于softirq和tasklet一样），因此真的休眠了，比如调用了会导致block的例程，内核几乎肯定会死。

2.	schedule()在切换进程时，保存当前的进程上下文（CPU寄存器的值、进程的状态以及堆栈中的内容），以便以后恢复此进程运行。中断发生后，内核会先保存当前被中断的进程上下文（在调用中断处理程序后恢复）；

但在中断处理程序里，CPU寄存器的值肯定已经变化了吧（最重要的程序计数器PC、堆栈SP等），如果此时因为睡眠或阻塞操作调用了schedule()，则保存的进程上下文就不是当前的进程context了.所以不可以在中断处理程序中调用schedule()。

3.	内核中schedule()函数本身在进来的时候判断是否处于中断上下文:

```cpp
if(unlikely(in_interrupt()))

BUG();
```

4.	中断handler会使用被中断的进程内核堆栈，但不会对它有任何影响，因为handler使用完后会完全清除它使用的那部分堆栈，恢复被中断前的原貌。

5.	处于中断context时候，内核是不可抢占的。因此，如果休眠，则内核一定挂起。

 



 

另一篇：

 

http://blog.openrays.org/blog.php?do=showone&tid=455

 

其结论：

 

5. 中断处理时可否睡眠问题Linux 设计中，中断处理时不能睡眠，这个内核中有很多保护措施，一旦检测到内核会异常。当 一个进程A因为中断被打断时，中断处理程序会使用 A 的内核栈来保存上下文，因为是“抢”的 A 的CPU，而且用了 A 的内核栈，因此中断应该尽可能快的结束。如果 do_IRQ 时又被时钟中断打断，则继续在 A 的内核栈上保存中断上下文，如果发生调度，则 schedule 进 switch_to，又会在
 A 的 task_struct->thread_struct 里保存此时时种中断的上下文。假如其是在睡眠时被时钟中断打断，并 schedule 的话，假如选中了进程 A，并 switch_to 过去，时钟中断返回后则又是位于原中断睡眠时的状态，抛开其扰乱了与其无关的进程A的运行不说，这里的问题就是：该如何唤醒之呢？？另外，和该中断共享中断号的中断也会受到影响。

 

======================================================

 

再一篇，也分析的很到位：

 

http://blog.csdn.net/maray/article/details/5770889

 

其结论：

Linux是以进程为调度单位的，调度器只看到进程内核栈，而看不到中断栈。在独立中断栈的模式下，如果linux内核在中断路径内发生了调度（从技术上讲，睡眠和调度是一个意思），那么linux将无法找到“回家的路”，未执行完的中断处理代码将再也无法获得执行机会。

————————————————————————————————————————————————————————————————
为什么软中断中也不能睡眠


    这个问题实际上是一个老生常谈的问题，答案也很简单，Linux在软中断上下文中是不能睡眠的，原因在于Linux的软中断实现上下文有可能是中断上下文，如果在中断上下文中睡眠，那么会导致Linux无法调度，直接的反应是系统Kernel Panic，并且提示dequeue_task出错。所以，在软中断上下文中，我们不能使用信号量等可能导致睡眠的函数，这一点在编写IO回调函数时需要特别注意。在最近的一个项目中，我们在dm-io的callback函数中去持有semaphore访问竞争资源，导致了系统的kernel
 panic。其原因就在于dm-io的回调函数在scsi soft irq中执行，scsi soft irq是一个软中断，其会在硬中断发生之后被执行，执行上下文为中断上下文。

 

       中断上下文中无法睡眠的原因大家一定很清楚，原因在于中断上下文不是一个进程上下文，其没有一个专门用来描述CPU寄存器等信息的数据结构，所以无法被调度器调度。如果将中断上下文也设计成进程上下文，那么调度器就可以对其进行调度，如果在开中断的情况下，其自然就可以睡眠了。但是，如果这样设计，那么中断处理的效率将会降低。中断（硬中断、软中断）处理都是些耗时不是很长，对实时性要求很高，执行频度较高的应用，所以，如果采用一个专门的后台daemon对其处理，显然并不合适。

 

       Linux对中断进行了有效的管理，一个中断发生之后，都会通过相应的中断向量表获取该中断的处理函数。在Linux操作系统中都会调用do_IRQ这个函数，在这个函数中都会执行__do_IRQ（），__do_IRQ函数调用该中断的具体执行函数。在执行过程中，该函数通过中断号找到具体的中断描述结构irq_desc，该结构对某一具体硬件中断进行了描述。在irq_desc结构中存在一条链表irqaction，这条链表中的某一项成员都是一个中断处理方法。这条链表很有意思，其实现了中断共享，例如传统的PCI总线就是采用共享中断的方法，该链表中的一个节点就对应了一个PCI设备的中断处理方法。在PCI设备驱动加载时，都需要注册本设备的中断处理函数，通常会调用request_irq这个函数，通过这个函数会构造一个具体的irq
 action，然后挂接到某个具体irq_desc的action链表下，实现中断处理方法的注册。在__do_IRQ函数中会通过handle_IRQ_event（）函数遍历所有的action节点，完成中断处理过程。到目前为止，中断处理函数do_IRQ完成的都是上半部的工作，也就是设备注册的中断服务程序。在中断上半部中，通常都是关中断的，基本都是完成很简单的操作，否则将会导致中断的丢失。耗时时间相对较长，对实时性要求不是最高的应用都会被延迟处理，都会在中断下半部中执行。所以，在中断上半部中都会触发软中断事件，然后执行完毕，退出服务。

 

       __do_IRQ完成之后，返回到do_IRQ函数，在该函数中调用了一个非常重要的函数irq_exit（），在该函数中调用invoke_softirq（），invoke_softirq调用do_softirq（）函数，执行软中断的操作。此时，程序的执行环境还是中断上下文，但是与中断上半部不同的是，软中断执行过程中是开中断的，能够被硬中断而中断。所以，如果用户的程序在软中断中睡眠，操作系统该如何调度呢？只有kernel panic了。另外，软中断除了上述执行点之外，还有其他的执行点，在内核中还有一个软中断的daemon处理软中断事务，驱动程序也可以自己触发一个软中断事件，并且在软中断的daemon上下文中执行。但是硬中断触发的事件都不会在这个daemon的上下文中执行，除非修改Linux中的do__IRQ代码。

 

       上述对软中断的执行做了简要分析，我对Linux中的硬中断管理机制做了一些代码分析，这一块代码量不是很大，可移植性非常的好~~建议大家阅读，对我上述的分析和理解存在什么不同意见，欢迎大家讨论。



#参考资料
-------

[中断--再问中断处理程序为什么不能睡眠 ?](http://bbs.chinaunix.net/thread-3778115-1-1.html)

[关于LINUX在中断（硬软）中不能睡眠的真正原因](http://blog.chinaunix.net/xmlrpc.php?r=blog/article&uid=22695386&id=196086)

[再思linux内核在中断路径内不能睡眠/调度的原因（2010）](https://blog.csdn.net/maray/article/details/5770889)

[]()

[]()

[]()


<br>

*	本作品/博文 ( [AderStep-紫夜阑珊-青伶巷草 Copyright ©2013-2017](http://blog.csdn.net/gatieme) ), 由 [成坚(gatieme)](http://blog.csdn.net/gatieme) 创作.

*	采用<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a><a rel="license"href="http://creativecommons.org/licenses/by-nc-sa/4.0/">知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议</a>进行许可. 欢迎转载、使用、重新发布, 但务必保留文章署名[成坚gatieme](http://blog.csdn.net/gatieme) ( 包含链接: http://blog.csdn.net/gatieme ), 不得用于商业目的.

*	基于本文修改后的作品务必以相同的许可发布. 如有任何疑问，请与我联系.