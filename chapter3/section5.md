# 3.5 回调地狱与 *Future* 对象

&emsp;&emsp;学习新东西，早晚得遇到坑。接下来我给大家介绍一个几乎所有人在用这种模型编程时都会遇到的坑，以及 *Vert.x* 提供的解决方法。

在使用 *Vert.x* 的异步无阻塞 API 时，如果我们要保证一系列操作的执行顺序，通常不能像一般的框架那样简单的依次调用，而是依次把要调用的方法放在前一个方法的事件处理函数中，用 *回调函数* 用的比较多的同事一定遇到这种情况：

```
String filePath = "X:\\path\\to\\file";
String backupPath = "X:\\path\\to\\backup\\folder";
Buffer buffer = Buffer.buffer("file content");

vertx.fileSystem().writeFile(filePath, buffer, write -> {
    if (write.succeeded()) {
        vertx.createNetClient().connect(1234, "localhost", connect -> {
            if (connect.succeeded()) {
                connect.result().sendFile(filePath, send -> {
                    connect.result().close();
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

* Vert.x Rx
* Future
* Vert.x Async
* Kotlin coroutine

&emsp;&emsp;其中，除 `Future` 外的其他三种都需要额外增加依赖，像 *Kotlin coroutine* 这个方法还是利用了 *Kotlin* 刚发布的新特性，稳定性、可靠性各方面还有待检验。我今天只重点介绍使用 Vert.x 核心中提供的 `Future` 来解决回调地狱的方法，大家如果对其它的方法感兴趣可以找官网的文档和例子看一下。

&emsp;&emsp;回到那个回调地狱的例子里，我们思考一下，如果让我们自己解决这个问题应该怎么办。

&emsp;&emsp;...

&emsp;&emsp;Future 是用来存储一个异步API调用结果的对象。它有未完成、已完成两种状态，其中已完成状态又包括成功和失败两种状态。在成功状态下，该对象保存调用结果，它可能是一个连接成功得到的Socket，也可能是一个打开成功得到的文件句柄；在失败状态下，该对象保存导致失败的异常；同时它还允许创建者设置一个 *回调函数* ，该回调函数会在它被完成时调用。

使用 Future 将回调地狱初步改造为：

```
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
    futureConnect.result().close();
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
&emsp;&emsp;这样代码就不会随着业务流程长度而无限制缩进了。不过这样还存在一些问题：

> 1. 颠倒两个代码块的顺序，该程序仍然是可以运行的，这看似灵活，但是缺乏一个顺序上的约束更容易产生混乱的代码；  
> 2. 异常处理都是重复代码。

&emsp;&emsp;所以，我们使用 `Future` 的 `compose` 方法再次重构这部分代码：

```
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
    futureConnect.result().close();
    vertx.fileSystem().copy(filePath, backupPath, futureCopy);
}, futureCopy).compose(v -> {
    vertx.fileSystem().delete(filePath, futureDelete);
}, futureDelete).setHandler(ar -> {
    if (ar.succeeded()) {
        logger.info("Hello, future compose!!!");
    } else {
        logger.error(ar.cause().getMessage());
    }
});
```
&emsp;&emsp;除了最后一个 *回调函数* ，前面的所有 *回调函数* 拿到的并不是一个 `AsyncResult<T>` 对象，而是我们期望的结果；也就是说它们只处理上一部成功的情况，失败的异常只需要在最后一步处理；这是不是有点像传统的 `try catch` 结构。 
&emsp;&emsp;好的，这几乎就是使用 `Future` 改造回调地狱的终极解决方案了，`compose` 方法还有另外一个重载实现，有兴趣的同事可以自己尝试写写看。



