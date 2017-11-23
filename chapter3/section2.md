# 3.2 Hello, Vert.x 与异步无阻塞 API

&emsp;&emsp;讲完了茴香豆的茴字的几种写法，我们来正式跟 *Vert.x* 打个招呼。下面是参考官网主页上的例子写的一段样例代码：
```
import io.vertx.core.AbstractVerticle;

public class MyHttpServerVerticle extends AbstractVerticle {
	
    private static final String[] ARRAY = new String[] {
        "May the Force be with you.",
        "Why so serious?",
        "Farewell, My Concubine."
    };

    private int index;
	
    @Override
    public void start() {
		
        index = 0;
		
        vertx.createHttpServer().requestHandler(req -> {
            req.response()
                .putHeader(HttpHeaders.CONTENT_TYPE, "text/plain")
                .end(ARRAY[index]);
        }).listen(8080);
		
        vertx.setPeriodic(1000, timerId -> {
            index = index + 1;
            index = index % ARRAY.length;
        });
    }
}
```
> 注：*pom.xml* 文件中要添加依赖：
> ```
> <dependency>
>     <groupId>io.vertx</groupId>
>     <artifactId>vertx-core</artifactId>
>     <version>3.5.0</version>
> </dependency>
> ```

&emsp;&emsp;这个例子首先创建了一个 *HTTP服务器*，为它设置了HTTP请求的 *处理函数*（Handler），并监听 *8080* 端口；在 *处理函数* 中，对于每个 *请求* 都直接返回一段文本；另外还设置了一个每1000毫秒触发一次的 *定时器* ，每次将文本的索引切换一下。

> 注：这个例子里的两个 *处理函数* 都是以 *Lambda表达式* 的方式给出的，它们的类型分别是 `Handler<HttpServerRequest>` 和 `Handler<Long>` 。

&emsp;&emsp;*Vert.x* 中的基本模块叫做 *Verticle*，最常见的用法是定义一个类继承 `AbstractVerticle`，然后重载它的 `start` 方法，有些时候还需要重载 `stop` 方法。在一个 *Verticle* 实例被 *部署*（*deploy*）时，它的 `start` 方法被调用一次，在它被 *反部署*（*undeploy*，或者叫 *撤销*）时，它的 `stop` 方法被调用一次。

&emsp;&emsp;在部署这个 *Verticle* 之前，我们首先需要实例化一个 `Vertx` 对象，像这样：
```
Vertx vertx = Vertx.vertx();
```
&emsp;&emsp;通常一个程序只需要实例化一个 `Vertx` 对象。然后，像这样部署一个Verticle：
```
vertx.deployVerticle(MyHttpServerVerticle.class.getName());
```
&emsp;&emsp;你还可以用这个 `Vertx` 实例部署更多 *Verticle* 类。如果你想获得部署结果，可以这样写：
```
vertx.deployVerticle(MyHttpServerVerticle.class.getName(), ar -> {
    if (ar.succeeded()) {
        System.out.println(ar.result());
    } else {
        System.err.println(ar.cause().getMessage());
    }
});
```
&emsp;&emsp;这里需要说明一下，这里 *处理函数*（或者说 *Lambda表达式*）的类型是 `Handler<AsyncResult<String>>`，所以参数类型是 `AsyncResult<String>`，`AsyncResult` 对象代表了一个异步操作的结果，在异步操作成功时，它保存了特定类型的结果，我们可以通过 `result` 方法获取这个结果；在异步操作失败时，它保存了导致失败的异常，我们可以通过 `cause` 方法获取这个异常。  
&emsp;&emsp;在上面的代码中，如果部署成功，部署生成的 `deploymentID` 会被打印出来，如果部署失败，会打印异常的描述信息。如果端口没被占用，我们的网站应该就发布成功了，在浏览器里打开

> `http://localhost:8080/`

即可看到返回的文本。

&emsp;&emsp;我们之前有提到Vert.x是事件驱动的，那么我们回顾一下之前写的代码，看看具体体现在哪。  
&emsp;&emsp;我们目前写的所有代码，除了个别直接在内存中构造对象的简单函数，所有需要访问网络、IO、可能耗费较长时间的操作都不是我们直接处理的，我们写的每一个事件处理函数都是在当前Verticle接收到一个相应事件（Event）后异步调用的，而且，一个Verticle中的所有事件处理函数（Handler）都是运行在一个事件循环（EventLoop）中，即完全的单线程。由此引入了下面的一些概念。
