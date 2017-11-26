# 1. 什么是 *Vert.x* ?

&emsp;&emsp;按照官网给的定义，**Eclipse Vert.x** 是一个在 **Java 虚拟机** 上用于构建 **响应式** 应用的 **工具包** 。

> &emsp;&emsp;Eclipse Vert.x is a tool-kit for building reactive applications on the JVM.

&emsp;&emsp;按照 “*响应式宣言（The Reactive Manifesto）*”（一份倡议性的文件） 里的定义，“响应式系统“是具有灵敏性、有回复性、可伸缩性、消息驱动等特点的计算机系统；而这里称之为*工具包*（而不是“框架”），是在强调 *Vert.x* 是*轻量*而高度*灵活*的。下面是它官网上宣称的一些特点，我只进行简单的搬运和翻译：

> 1. 事件驱动，无阻塞。这意味着（*为什么?*）你的应用可以用少量的内核线程处理大量并发，节省了硬件资源；
> 2. *Vert.x* 支持多语言，包括 *Java*、*JavaScript*、*Groovy*、*Ruby*、*Ceylon*、*Scala* 和 *Kotlin* 。官方并不会给你安利 *什么是最好的语言* ，而是希望使用者根据项目需要以及团队技术储备进行选择；
> 3. 通用性强。开发简单的网络应用、复杂的 Web 应用、HTTP/RESTful微服务、大量的事件处理或后台消息服务，*Vert.x* 都是很合适的选择；
> 4. 不自以为是（unopinionated）。*Vert.x* 不是一个严格定义的框架或者容器。官方不会告诉你编写一个应用的正确姿势，只给你一些有用的组件、文档和例子；
> 5. *Vert.x* 很有趣，简约而不简单...blahblah...

&emsp;&emsp;总结一下，*Vert.x* 是一个 *高效的、轻量的、灵活的、支持多语言的、通用性强的工具包*，如图（误）：

<center><img src="sak.jpg"></img></center>
