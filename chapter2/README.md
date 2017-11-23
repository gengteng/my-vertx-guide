# 2. 为什么要用 *Vert.x* ?

&emsp;&emsp;这一段的内容其实还是搬运，不过不是搬运自Vert.x官网，而是它的一个同类框架Akka的官网文档，主要有两篇文章：

> &emsp;&emsp;[1. Why modern systems need a new programming model](https://doc.akka.io/docs/akka/current/guide/actors-motivation.html)

> &emsp;&emsp;[2. How the Actor Model Meets the Needs of Modern, Distributed Systems](https://doc.akka.io/docs/akka/current/guide/actors-intro.html)

&emsp;&emsp;第一篇文章 *提出了一些现存的问题* ，第二篇文章讲 *参与者模型* （也就是 *Vert.x* 和 *Akka* 实现的模型）是 *如何解决这些问题* 的，我给大家依次总结下要点，有兴趣的同事可以自己去看一下。

&emsp;&emsp;第一篇先讲了传统 *Java* 面向对象的多线程模型的缺点：

1. *锁* 在现代 CPU 架构中是开销比较大的，严重限制了高并发；

2. 调用线程可能会被 *阻塞* ，存在无法工作的情况，对于服务器端开发尤其浪费资源，开更多的线程仍然是一个消耗；

3. 锁可能导致 *死锁* 。

&emsp;&emsp;然后介绍了关于共享内存的几点问题：

1. 并不存在真正意义的共享内存，CPU的多个核心会把大量数据传来传去。

TODO...
