# 3.4 事件总线与集群化

&emsp;&emsp;到目前为止，我们的介绍都是围绕着单个 *Verticle* ，但是一个项目通常不止一个模块，所以问题来了：—

> 如果有多个 *Verticle*，它们之间怎么通信呢？

&emsp;&emsp;首先，我想定义一下“多个 *Verticle*”。我们这里说的“多个 *Verticle*”指的是我们实现的多个不同的 *Verticle* 类，区别于同一个 *Verticle* 类部署成多个实例——是的，这一点我前面没有讲，一个 *Verticle* 是单线程的，但是如果某个 *Verticle* 类对并发有更高的要求，想利用多核 CPU 的话，可以像这样部署：

```java
vertx.deployVerticle(MyHttpServerVerticle.class.getName(), 
    new DeploymentOptions().setInstances(8));
```
&emsp;&emsp;这段代码会为 `MyHttpServerVerticle` 部署 8 个实例，分别跑在 8 个线程上；我们不用担心它们监听同一个端口会有冲突， *Vert.x* 会为它们做分发处理。通常这种情况，这个实例数应当设置为 `Runtime.getRuntime().availableProcessors() * 2`（为什么？）。

&emsp;&emsp;上面插了一段单个 *Verticle* 利用多核 CPU 横向扩展的介绍，下面开始回答关于 “真 · 多个 *Verticle*” 的通信问题。  
&emsp;&emsp;*Vert.x* 模块间通信使用的是 *事件总线（EventBus）* 。在每一个 *Vert.x* 实例中，都存在一个唯一的 *事件总线（EventBus）* 实例，可以通过 `vertx.eventBus()` 获得。作为 ***Vert.x* 的神经系统**，*事件总线* 允许应用的各个部分——无论这些部分是用哪种语言写的、是否在一个 *Vert.x* 实例中——都可以通过它进行通信。

&emsp;&emsp;// TODO