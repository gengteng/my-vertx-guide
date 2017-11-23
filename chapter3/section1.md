# 3.1 *Java 8* 的 *Lambda表达式*

&emsp;&emsp;*Vert.x*从版本3（当前最新*Vert.x*版本为3.5.0）开始就需要使用*Java 8*,并且大量使用了*Lambda表达式*这个*Java 8*的新特性。所以在真正介绍怎么使用*Vert.x*之前，我先介绍一下*Java 8*中的*Lambda表达式*。

&emsp;&emsp;*Lambda表达式*在很多语言中都有，其本质就是匿名函数：

JavaScript：
```
function (a, b) {  
    return a + b; 
};
```
Python：
```
lambda a,b: a + b
```
Lua：  
```
function (a, b) 
    return a + b 
end
```
C++： 
```
[](int a, int b) {
    return a + b;
};
```

C#：  
```
(a, b) => a + b;
```
&emsp;&emsp;*Lambda表达式* 在 *Python*、*JavaScript*、*Lua* 等弱类型语言中可以直接作为一个普通变量，在*C++*中可以作为特定类型的函数指针（使用auto作为指针变量的类型即可自动推断类型），在 *C#* 可以作为一个特定类型的委托变量，而在 *Java 8* 中，*Lambda表达式* 实际上是一个符合特定条件的 *接口* 的实现。其实在过去版本的 *Java* 中，如果我们想像传递变量那样传递一个 *函数* （或者叫 *方法* ），通常是先定义一个 *接口* ，像这样：
```
public interface MyFunction { 
    int exec(int a, int b);  
}
```
&emsp;&emsp;然后，我们再实现这个接口，像这样：
```
MyFunction add = new MyFunction () { 
    @Override
    public int exec(int a, int b) {
        return a + b; 
    }
};
```
&emsp;&emsp;这个 `add` 变量（或者叫*对象*）的 `exec` 就是我们要传递的函数，我们可以通过传递 `add` 变量来传递这个函数。现在，在*Java 8*中，你可以不用修改接口定义，直接用 *Lambda表达式* 这样写：
```
MyFunction add = (a, b) -> a + b;
```
&emsp;&emsp;也可以这样写：
```
MyFunction add = (int a, int b) -> a + b;
```
&emsp;&emsp;或者这样写：
```
MyFunction add = (a, b) -> { 
    return a + b; 
};
```
&emsp;&emsp;当然以前的写法也是完全可以的，不过我觉得能一行搞定的事你肯定不会用四行了。不过配合Lambda表达式使用的接口需要满足一个条件：

> **除静态方法和默认方法外，该接口只有一个抽象方法。**

&emsp;&emsp;这是*一个接口能用来定义一个Lambda表达式变量*的充分必要条件，而那个抽象方法名是什么并不重要（你都是我的唯一了，我还能找不到你?）。这样的接口也被称为函数接口。所以，*Java*原有的很多只有单个抽象方法的接口都可以写成Lambda表达式了，比如 `Runnable`，以前我们启动一个新线程运行一段代码会这样写：
```
new Thread(new Runnable() { 
    @Override 
    public void run() {  
        // blah...blah...  
    } 
}).start();
```
&emsp;&emsp;而现在，我们可以这样写：
```
new Thread(() -> {
    // blah...blah... 
}).start();
```
&emsp;&emsp;没错，只要你写的*Lambda表达式*类型正确就可以了，下面是一个错误的例子，无法通过编译，因为 `Runnable` 的 `run` 方法是没有参数的，应当写一对小括号而不是一个变量名 `arg`：
```
new Thread(arg -> { 
    // blah...blah... 
}).start();
```
&emsp;&emsp;*Java 8*还提供了一个注解，`@FunctionalInterface` ，这个注解用来加在一个函数接口定义上，它会检查上面那个*充分必要条件*，如果不满足，比如：
```
@FunctionalInterface
public interface MyFunction {
    int execute(int a, int b);
    int wtf(int i);
}
```
&emsp;&emsp;编译器将会报错：
> MyFunction is not a functional interface.

&emsp;&emsp;这个注解仅仅是一种约束和对调用者的提示，虽然函数接口不写这个注解仍然可以用做 *Lambda表达式* 的类型，但是如果这个接口的确是一个函数接口，那么写上是个比较好的习惯。大家可以看一下 *Java* 原有的 `Runable`、`Callable`、`Comparator` 接口，都已经加上了 `@FunctionalInterface` 注解。在 *Vert.x* 中，通常不需要你自己定义函数接口，基本上只需要用到它定义好的 *事件处理函数* （以下简称 *处理函数*）的接口 `Handler<E>`：

```
@FunctionalInterface
public interface Handler<E> {
    void handle(E event);
}
```
&emsp;&emsp;另外，*Java 8* 本身也提供了 `Consumer、Predicate、Function` 等很多函数接口。







