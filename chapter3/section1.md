# 3.1 *Java 8* 的 *Lambda表达式*

&emsp;&emsp;*Vert.x*从版本3（当前最新*Vert.x*版本为3.5.0）开始就需要使用*Java 8*,并且大量使用了*Lambda表达式*这个*Java 8*的新特性。所以在真正介绍怎么使用*Vert.x*之前，我们需要先了解一下*Java 8*中的*Lambda表达式*。

&emsp;&emsp;*Lambda表达式*在很多语言中都有，其本质就是匿名函数：

JavaScript：
```javascript
function (a, b) {  
    return a + b; 
};
```
Python：
```python
lambda a,b: a + b
```
Lua：  
```lua
function (a, b) 
    return a + b 
end
```
C++： 
```cpp
[](int a, int b) {
    return a + b;
};
```
C#：  
```C#
(a, b) => a + b;
```
&emsp;&emsp;*Lambda表达式* 在 *Python*、*JavaScript*、*Lua* 等弱类型语言中可以直接作为一个普通变量，在 <em>C++</em> 中可以作为特定类型的函数指针（使用auto作为指针变量的类型即可自动推断类型），在 *C#* 中可以作为一个特定类型的委托变量，而在 *Java 8* 中，*Lambda表达式* 实际上就是一个 *匿名内部类*。

&emsp;&emsp;什么是 *匿名内部类* 呢？在过去版本的 *Java* 中，如果我们想像传递变量那样传递一个 *函数* （或者叫 *方法* ），通常是先定义一个 *接口* ，像这样：
```java
public interface MyFunction { 
    int exec(int a, int b);  
}
```
&emsp;&emsp;然后，我们再实现这个接口，像这样：
```java
MyFunction sum = new MyFunction () { 
    @Override
    public int exec(int a, int b) {
        return a + b; 
    }
};
```
&emsp;&emsp;这个 `sum` 变量（或者叫*对象*）的 `exec` 就是我们要传递的 *函数*，我们可以通过传递 `sum` 变量来传递这个 *函数*，而这个对接口的实现就叫做 *匿名内部类*。

&emsp;&emsp;现在，在*Java 8*中，你可以不用修改接口定义，直接用 *Lambda表达式* 这样写：
```java
MyFunction sum = (a, b) -> a + b;
```
&emsp;&emsp;也可以这样写：
```java
MyFunction sum = (int a, int b) -> a + b;
```
&emsp;&emsp;或者这样写：
```java
MyFunction sum = (a, b) -> { 
    return a + b; 
};
```
&emsp;&emsp;当然以前的写法仍然是可以的，不过我觉得能一行搞定的事你肯定不会再用四行了。与一般的 *匿名内部类* 所实现的接口不同的是，配合 *Lambda表达式* 使用的接口（比如这里的 `MyFunction`）需要满足一个条件：

> **除静态方法和默认方法外，该接口只有一个抽象方法。**

&emsp;&emsp;这个条件是*一个接口能用做一个Lambda表达式的类型*的充分必要条件，而那个抽象方法名是什么并不重要。这样的接口也被称为 *函数接口*。所以，按照这个条件，*Java*原有的很多只有单个抽象方法的接口都可以写成Lambda表达式了，比如 `Runnable`，以前我们启动一个新线程运行一段代码会这样写：
```java
new Thread(new Runnable() { 
    @Override 
    public void run() {  
        // blah...blah...  
    } 
}).start();
```
&emsp;&emsp;而现在，我们可以这样写：
```java
new Thread(() -> {
    // blah...blah... 
}).start();
```
&emsp;&emsp;只要你写的*Lambda表达式*类型正确就可以了，下面是一个错误的例子，无法通过编译，因为 `Runnable` 的 `run` 方法是没有参数的，应当写一对小括号而不是一个变量名 `arg`：
```java
new Thread(arg -> { 
    // blah...blah... 
}).start();
```
&emsp;&emsp;另外，*Java 8*还提供了注解 `@FunctionalInterface` ，用来加在一个 *函数接口* 的定义上。它会检查上面那个*充分必要条件*，如果不满足，比如：
```java
@FunctionalInterface
public interface MyFunction {
    int execute(int a, int b);
    int wtf(int i);
}
```
&emsp;&emsp;编译器将会报错：
> MyFunction is not a functional interface.

&emsp;&emsp;这个注解仅仅是一种约束和对调用者的提示，虽然函数接口不写这个注解仍然可以用做 *Lambda表达式* 的类型，但是如果这个接口的确是一个函数接口，那么写上是个比较好的习惯。大家可以看一下 *Java* 原有的 `Runable`、`Callable`、`Comparator` 接口，在 *Java 8* 中都加上了 `@FunctionalInterface` 注解。在 *Vert.x* 中，通常不需要你自己定义函数接口，基本上只需要用到它定义好的 *事件处理函数* （以下简称 *处理函数*）的接口 `Handler<E>`：

```java
@FunctionalInterface
public interface Handler<E> {
    void handle(E event);
}
```
&emsp;&emsp;加上 *Java 8* 本身也提供了 `Consumer<T>`、`Predicate<T>`、`Supplier<T>`、`Function<T,R>` 等很多 *函数接口*，我们基本不用自己定义 *函数接口*。

&emsp;&emsp;最后，再介绍一个与 *Lambda表达式* 相关的 *Java 8* 新特性——η转换（eta-conversion）——一个 *函数接口* 对象还可以写成下面的形式：
```java
public class EtaConversion {

    public static void main(String[] args) {

        int a = 4;
        int b = 3;
        
        // sum 非静态方法需要通过实例获得
        EtaConversion obj = new EtaConversion();
        int c = ofSquare(obj::sum, a, b);

        // diff 静态方法通过类获得即可
        int d = ofSquare(EtaConversion::diff, a, b);

        // 打印计算结果
        System.out.println(a + " 和 " + b + " 的平方和为 " + c + "，平方差为 " + d + "。");
    }

    // 先对两个值分别求平方，再按 func 计算
    public static int ofSquare(MyFunction func, int a, int b) {
        return func.exec(a * a, b * b);
    }

    // 求和
    public int sum(int a, int b) {
        return a + b;
    }

    // 求差（静态方法）
    public static int diff(int a, int b) {
        return a - b;
    }
}
```
&emsp;&emsp;以上，就是我要分享的关于 *Java 8* 的新特性 *Lambda表达式* 的所有内容。







