# 3.6 自定义异步方法

&emsp;&emsp;最后，我介绍一下 *如何组合 Vert.x 提供异步 API 实现自己的异步方法*。

&emsp;&emsp;我们通常会自定义一些像下面这样方法，它的结果将通过返回值得到：
```java
Result getSomething(Argument ...args);
```
&emsp;&emsp;在调用时我们会用 `try` 把它包裹起来，以便在无法得到结果的时候处理异常:
```java
try {
    Result result = getSomething(args);
    System.out.println("Result is " + result.toString());
} catch (Exception e) {
    System.err.println(e.getMessage());
}
```
&emsp;&emsp;现在，我准备使用 Vert.x API 自定义一个根据主键从数据库特定表中取出一行数据的方法，它包括两个异步操作：

1. 从连接池中取出一个连接；
2. 用它查询一条记录并返回；

&emsp;&emsp;我们开始尝试使用 Vert.x 提供的 API 自定义一个这样方法，如下：
```java
public ResultSet getRecord(SQLClient client, String primaryKey) {

    sqlClient.getConnection(conn -> {
        if (conn.succeeded()) {
            SQLConnection connection = conn.result();
            connection.query("SELECT ID, DATA FROM RECORD WHERE ID='" + pk + "'", rec -> {
                connection.close(); // 一定要将不使用的连接放回连接池
                if (rec.succeeded()) {
                    ResultSet resultSet = rec.result();
                    // 成功，需要 return
                } else {
                    // 失败，需要 throw exception
                }
            });
        } else {
            // 失败，需要 throw exception
        }
    });
    
    // return ??;
}
```
&emsp;&emsp;到这我们已经得到查询结果了，问题是：结果怎么返回呢？如果 `return` 写在 *Lambda表达式* 里，返回的是那个 *Lambda表达式*，并不是我们的 `getRecord`；写在最后的话，按照异步调用的顺序，好像那时候结果还没查到；异常怎么抛出呢？在 *Lambda表达式* 里抛好像是抛到 *事件循环（EventLoop）* 里去了，不是从我们的 `getRecord` 抛出去，外面 `catch` 不到了。
&emsp;&emsp;我想大概很多同事已经知道该怎么实现了，不过这种初到 *异步世界* 的水土不服卡住过很多人。一种正确的实现方法如下：
```java
public void getRecord(SQLClient client, String primaryKey, Handler<AsyncResult<ResultSet>> handler) {
    client.getConnection(conn -> {
        if (conn.succeeded()) {
            SQLConnection connection = conn.result();
            connection.query("SELECT ID, DATA FROM RECORD WHERE ID='" + primaryKey + "'", rec -> {
                connection.close(); // 一定要将不使用的连接放回连接池
                
                handler.handle(rec);
            });
        } else {
            handler.handle(Future.failedFuture(conn.cause()));
        }
    });
}
```
&emsp;&emsp;这样调用：
```java
getRecord(client, primaryKey, ar -> {
    if (ar.succeeded()) {
        ResultSet record = ar.result();
        // TODO: 得到 record，继续处理
    } else {
        System.err.println(ar.cause().getMessage()); // 打印异常信息
    }
});
```
&emsp;&emsp;另一种正确的实现方法如下：
```java
public Future<ResultSet> getRecord(SQLClient client, String primaryKey) {
    Future<ResultSet> futureRecord = Future.future();

    client.getConnection(conn -> {
        if (conn.succeeded()) {
            SQLConnection connection = conn.result();
            connection.query("SELECT ID, DATA FROM RECORD WHERE ID='" + primaryKey + "'", rec -> {
                connection.close(); // 一定要将不使用的连接放回连接池
                
                futureRecord.handle(rec);
            });
        } else {
            futureRecord.fail(conn.cause());
        }
    });

    return futureRecord;
}
```
&emsp;&emsp;这样调用：
```java
getRecord(client, primaryKey).setHandler(ar -> {
    if (ar.succeeded()) {
        ResultSet record = ar.result();
        // TODO: 得到 record，继续处理
    } else {
        // 打印异常信息
        System.err.println(ar.cause().getMessage());
    }
});
```
&emsp;&emsp;或者:
```java
getRecord(client, primaryKey).compose(resultSet -> {
    // TODO: 得到 record，继续处理
}, nextFuture).compose(nextResult -> {

...... // 若干次 compose 组成的链式调用

}, finalFuture).setHandler(ar -> {
    if (ar.succeeded()) {
        // 这里得到的是整个链式调用的结果，不是 getRecord 的结果
    } else {
        // 如果 getRecord 失败，会跳到这里打印异常
        System.err.println(ar.cause().getMessage()); 
    }
});
```
&emsp;&emsp;所以，我们使用 *Vert.x* 的异步 API 自定义方法时，应当实现为像:
```java
void getRecord(SQLClient client, String primaryKey, Handler<AsyncResult<ResultSet>> handler);
```
&emsp;&emsp;或
```java
Future<ResultSet> getRecord(SQLClient client, String primaryKey);
```
&emsp;&emsp;这样的形式，也正是 *函数式编程* 中 *高阶函数* 的两种典型的形式。

> 注：*高阶函数* 是指接收另外一个函数作为参数，或返回一个函数的函数。