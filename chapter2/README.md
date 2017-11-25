# 2. 为什么要用 *Vert.x* ?

&emsp;&emsp;这一段的内容其实还是搬运，不过不是搬运自 *Vert.x* 官网，而是它的一个同类框架 *Akka* 的官网文档，主要有两篇文章：

> &emsp;&emsp;1. [Why modern systems need a new programming model](https://doc.akka.io/docs/akka/current/guide/actors-motivation.html)

> &emsp;&emsp;2. [How the Actor Model Meets the Needs of Modern, Distributed Systems](https://doc.akka.io/docs/akka/current/guide/actors-intro.html)

&emsp;&emsp;第一篇文章提出了一些当代计算机系统在开发多线程应用时的问题 ，第二篇文章主要介绍 *参与者模型* （Actor Model，也就是 *Vert.x* 和 *Akka* 实现的模型）是 *如何解决这些问题* 的。我给大家总结下要点，有兴趣的同事可以自己仔细阅读一下。

&emsp;&emsp;第一篇首先指出了传统 *Java* 面向对象编程中的通过使用锁来解决多线程资源问题的缺点：

1. *锁* 在现代 CPU 架构中是开销比较大的，严重限制了高并发；

2. 调用线程可能会被 *阻塞* ，存在无法工作的情况，对于服务器端开发尤其浪费资源，开更多的线程仍然是一个消耗；

3. 锁可能导致 *死锁* 。

&emsp;&emsp;然后介绍了关于共享内存的几点问题：

1. 并不存在真正意义的共享内存，为了在 CPU 的多个核心共享数据（比如被声明为 `volatile` 的变量），它们会把大量 CPU 缓存数据传来传去；
2. 相比使用共享内存或原子化（atomic）的数据结构，在并发实体中维护自身的状态以及显式地通过消息来传递数据可能是更好的方法。（对 *Golang* 比较了解的同事应该知道，这有点像 *Golang* 所宣称的某种哲学：不要通过共享内存来通信，而应该通过通信来共享内存。）

&emsp;&emsp;最后，总结了现代计算机为了实现高并发与高性能，使用调用后台线程处理任务时存在的两个问题：

1. 基于调用栈的异常处理不再适用，应当引入显式的异常信号机制；
2. 调用者线程应当及时地获得被调线程的处理结果。

&emsp;&emsp;以上就是第一篇文章提出的所有问题。

&emsp;&emsp;然后，在第二篇文章中，作者提出了通过传递消息来避免 *锁* 与 *阻塞*以及“优雅地”处理错误的方法，这里不进行详细的介绍。在介绍如何使用 *Vert.x* 的时候，我会在适当的时候对具体原理进行解释。
