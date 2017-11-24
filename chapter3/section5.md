# 3.5 回调地狱与 *Future* 对象

&emsp;&emsp;什么技术方案都是有坑的。接下来我给大家介绍一个几乎所有人在用这种模型编程时都会遇到的坑，以及 *Vert.x* 提供的解决方法。

在使用 *Vert.x* 的异步无阻塞 API 时，如果我们要保证一系列操作的执行顺序，通常不能像一般的框架那样简单的依次调用，而是依次把要调用的方法放在前一个方法的事件处理函数中，用 *回调函数* 用的比较多的同事一定遇到这种情况：

```java
String filePath = "X:\\path\\to\\file";
String backupPath = "X:\\path\\to\\backup\\folder";
Buffer buffer = Buffer.buffer("file content");

vertx.fileSystem().writeFile(filePath, buffer, write -> {
    if (write.succeeded()) {
        vertx.createNetClient().connect(1234, "localhost", connect -> {
            if (connect.succeeded()) {
                connect.result().sendFile(filePath, send -> {
                    connect.result().close(); // 关闭不再使用的Socket
                    if (send.succeeded()) {
                        vertx.fileSystem().copy(filePath, backupPath, copy -> {
                            if (copy.succeeded()) {
                                vertx.fileSystem().delete(filePath, delete -> {
                                    if (delete.succeeded()) {
                                        logger.info("Hello, callback hell.");
                                    } else {
                                        logger.error(delete.cause().getMessage());
                                    }
                                });
                            } else {
                                logger.error(copy.cause().getMessage());
                            }
                        });
                    } else {
                        logger.error(send.cause().getMessage());
                    }
                });
            } else {
                logger.error(connect.cause().getMessage());
            }
        });
    } else {
        logger.error(write.cause().getMessage());
    }
});
```

&emsp;&emsp;这段代码先是把一段内容写到一个新文件里，然后建立一个 *TCP* 连接把文件发过去，再把这个文件拷贝到另一个目录作为备份，最后把原文件删掉。回调函数一层层的嵌套，形成了这样的代码结构，这就是回调地狱。

&emsp;&emsp;如果是嵌套个两三层，其实还可以接受，如果业务流程比较长，这样的代码就很难看了。*Vert.x* 提供了四种方法解决这个问题：

* Future
* Vert.x Rx
* Vert.x Async
* Kotlin coroutine

&emsp;&emsp;其中，除 `Future` 外的其他三种都需要额外增加依赖，其中 *Vert.x Rx* 是使用观察者模式实现的响应式接口，而 *Vert.x Async* 和 *Kotlin coroutine* 则是利用了“协程（Fiber/Coroutine）” 来实现像调用同步阻塞接口那样使用异步无阻塞接口。我今天只重点介绍使用 Vert.x 核心中提供的 `Future` 来解决回调地狱的方法，大家如果对其它的方法感兴趣可以找官网的文档和例子看一下。

&emsp;&emsp;我们直接看 Future 的定义，下面只列出与我们要将的内容有关的部分：

```java
public interface Future<T> extends AsyncResult<T>, Handler<AsyncResult<T>> {

    // 用结果 result 将这个未完成的 AsyncResult<T> 设为成功
    void complete(T result);

    // 用异常 cause 将这个未完成的 AsyncResult<T> 设为失败
    void fail(Throwable cause);

    // 设置该 Future 对象被完成时应该调用的处理函数
    Future<T> setHandler(Handler<AsyncResult<T>> handler);

    // 其他方法
}
```

&emsp;&emsp;它是一个泛型接口，继承自 *异步结果* `AsyncResult<T>` 和 *异步结果处理函数* `Handler<AsyncResult<T>>`；我们前面介绍过 *异步结果* 对象可以保存异步操作的状态（未完成、成功或失败），以及提供结果和异常的获取方法，`Future<T>` 继承自它，表明 `Future<T>` 也有这些特性；同时，不同于只能读取状态和结果的 `AsyncResult<T>`，它还提供了 `complete` 方法和 `fail` 方法，使其可以从外部被完成；

&emsp;&emsp;但是，它同时还继承自 *异步结果处理函数* `Handler<AsyncResult<T>>` 是怎么回事？一个 `Future<T>` 对象同时还是一个函数？它作为一个函数，要处理的这个 *异步结果* `AsyncResult<T>` 又是哪来的呢？

我在 `Future<T>` 接口的实现 `FutureImpl<T>` 类里找到了这个：

```java
    @Override
    public void handle(AsyncResult<T> asyncResult) {
        if (asyncResult.succeeded()) {
            complete(asyncResult.result());
        } else {
            fail(asyncResult.cause());
        }
    }
```
&emsp;&emsp;没错，这里是它作为一个 *处理函数* `Handler<AsyncResult<T>>` 的实现，当它被调用时，就是在处理另一个 *异步结果* `AsyncResult<T>`，用另一个 *异步结果* 的状态和值来完成作为 *异步结果* `AsyncResult<T>` 的自己。

&emsp;&emsp;我们再看一下刚刚在 *回调地狱* 里用的那几个 *Vert.x* 提供的 API 的定义：
```java
    // 写一段内容到新文件里
    FileSystem writeFile(String path, Buffer data, Handler<AsyncResult<Void>> handler);

    // 建立与某个地址的Socket连接
    NetClient connect(int port, String host, Handler<AsyncResult<NetSocket>> connectHandler);

    // 使用Socket发送一个文件
    NetSocket sendFile(String filename, Handler<AsyncResult<Void>> resultHandler)

    // 复制一个文件到另一个位置
    FileSystem copy(String from, String to, Handler<AsyncResult<Void>> handler);

    // 删除一个文件
    FileSystem delete(String path, Handler<AsyncResult<Void>> handler);
```
&emsp;&emsp;我们还可以继续罗列更多的 *Vert.x* 提供的 API，只要可以，它们的最后一个参数总是 `Handler<AsyncResult<T>>`，所以我们按照文档中的方法实例化一个 `Future` 对象：
```java
Future<Void> futureWrite = Future.future();
```
&emsp;&emsp;然后，我们调用那个写文件的方法，不过不再使用 *Lambda表达式*，而是把 `futureWrite` 放到原来 *Lambda表达式* 的位置：
```java
vertx.fileSystem().writeFile(filePath, buffer, futureWrite);
```
&emsp;&emsp;如果你还记得 `Future<T>` 继承自 `Handler<AsyncResult<T>>` ，应该不会惊讶于这样的写法。然后，我们给 `futureWrite` 设置一个它在完成时应该调用的 *处理函数*：
```java
futureWrite.setHandler(ar -> {
    if (ar.succeeded()) {
        // success
    } else {
        // error
    }
});
```
&emsp;&emsp;这样就实现了读取文件后 `futureWrite` 对象被完成，`futureWrite` 对象的 *处理函数* 被调用的逻辑。

&emsp;&emsp;这样真的有助于解决回调地狱吗？我们可以尝试继续使用这种方法改造上面的回调地狱：

```java
Future<Void> futureWrite = Future.future();
Future<NetSocket> futureConnect = Future.future();
Future<Void> futureSend = Future.future();
Future<Void> futureCopy = Future.future();
Future<Void> futureDelete = Future.future();

vertx.fileSystem().writeFile(filePath, buffer, futureWrite);

futureWrite.setHandler(ar -> {
    if (ar.succeeded()) {
            vertx.createNetClient().connect(1234, "localhost", futureConnect);
    } else {
        logger.error(ar.cause().getMessage());
    }
});

futureConnect.setHandler(ar -> {
    if (ar.succeeded()) {
        ar.result().sendFile(filePath, futureSend);
    } else {
        logger.error(ar.cause().getMessage());
    }
});

futureSend.setHandler(ar -> {
    futureConnect.result().close(); // 关闭不再使用的Socket
    if (ar.succeeded()) {
        vertx.fileSystem().copy(filePath, backupPath, futureCopy);
    } else {
        logger.error(ar.cause().getMessage());
    }
});

futureCopy.setHandler(ar -> {
    if (ar.succeeded()) {
        vertx.fileSystem().delete(filePath, futureDelete);
    } else {
        logger.error(ar.cause().getMessage());
    }
});

futureDelete.setHandler(ar -> {
    if (ar.succeeded()) {
        logger.info("Welcome to the future!!!");
    } else {
        logger.error(ar.cause().getMessage());
    }
});
```
&emsp;&emsp;看起来效果还不错，整齐多了，代码也不会随着业务流程长度而无限制缩进了。不过这样还是存在一些问题：

> 1. 颠倒两个代码块的顺序，该程序仍然是可以运行的，这样一来没有顺序上的约束，很容易产生混乱的代码；  
> 2. 异常处理存在大量重复代码。

&emsp;&emsp;`Future` 还提供了一个用于链式调用的方法 `compose`，我们使用 `Future` 的 `compose` 方法再次重构这部分代码： 
```java
Future<Void> futureWrite = Future.future();
Future<NetSocket> futureConnect = Future.future();
Future<Void> futureSend = Future.future();
Future<Void> futureCopy = Future.future();
Future<Void> futureDelete = Future.future();

vertx.fileSystem().writeFile(filePath, buffer, futureWrite);

futureWrite.compose(v -> {
    vertx.createNetClient().connect(1234, "localhost", futureConnect);
}, futureConnect).compose(socket -> {
    socket.sendFile(filePath, futureSend);
}, futureSend).compose(v -> {
    futureConnect.result().close(); // 关闭不再使用的Socket
    vertx.fileSystem().copy(filePath, backupPath, futureCopy);
}, futureCopy).compose(v -> {
    vertx.fileSystem().delete(filePath, futureDelete);
}, futureDelete).setHandler(ar -> {
    if (ar.succeeded()) {
        logger.info("Hello, future compose!!!");
    } else {
        if (futureConnect.succeeded()) {
            futureConnect.result().close(); // 关闭不再使用的Socket
        }
        logger.error(ar.cause().getMessage());
    }
});
```
&emsp;&emsp;除了最后一个 *回调函数* ，前面的所有 *回调函数* 的参数并不是一个 `AsyncResult<T>` 对象，而是我们期望的结果，即一个类型为 `T` 的对象；也就是说每次 `compose` 只处理上一步成功的情况，失败的异常会被传递到最后一个 *回调函数* 处理；这是不是有点像传统的 `try catch` 结构。  
&emsp;&emsp;好的，这几乎就是使用 `Future` 改造回调地狱的终极解决方案了，`compose` 方法还有另外一个重载实现，有兴趣的同事可以自己尝试写写看。
