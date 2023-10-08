---
title: Kotlin Lambda
date: 2023-01-13 21:29:13
tags: [kotlin, lambda]
---

介绍Kotlin的特性时，往往喜欢拿Java来做对比  ~~（Java：老了被嫌弃了~~

在Java中，似乎只有引用类型的实例才称得上对象，基础类型和函数都最多只能算“二等公民”。
设计基础类型有一定的好处，即基本类型不需要实例化便可以使用，相比于与之对应的包装类型更加节省内存。
不过，Kotlin去除了基本类型这一设计。
函数不算做对象只能说是Java的设计缺陷，想要实现其他语言类似的效果，可以通过SAM间接传递函数引用。

```java
// SAM
interface OneParamRunnable<T> {
    void run(T t);
}

class Main {
    public static void log(Object o, OneParamRunnable<Object> runnable) {
        Logger.log("OneParam Runnable: ” + o.toString()");
        runnable.run(o);
    }

    public static void main(String[] args) {
        // 利用Java8简化Lambda的调用：
        Main.log(16, (i) -> perf.push(i));
    }
}
```

调用上的确比Java 8之前要**优雅**很多了，不过他还是有不少痛点：

1. 语言缺陷。没有单独的函数类型，需要借助SAM实现；
2. 声明冗余。参数以及返回值相同的SAM不能相互替代，不能彼此兼容；
3. 不够DSL。调用上不符合DSL风格。

让我们来看看，Kotlin是如何解决这些痛点的：

## Kotlin函数类型

在Kotlin中，可以将现有函数通过函数引用的方式进行传递，这与Java后来更新的函数引用类似。例如，某函数需要传入一个接收一个Int参数并返回一个String参数的Lambda时：

```kotlin
// Kotlin
typealias Converter = (Int) -> String

// …
fun onAction(i: Int, c: Converter) = c(i)
// 这里即为函数引用
onAction(1, Int::toString)
```

> 这里的类型别名不是必要的

也许你注意到了，Kotlin中不需要SAM，因为函数在Kotlin中有了自己的类型，定义需要使用到箭头，
箭头左侧表示参数列表，箭头右侧表示返回值，参数列表需要用括号包裹，多个参数需要在括号内用逗号分割。
上面的`(Int) -> String`代表参数列表为一个Int，返回值为String的函数。 
那么，你能猜出“参数列表为Int和Int，返回值为Int”的函数类型怎么表示吗？
另外，由于函数是类型，所以函数也能作为函数类型的参数或返回值，如：

```kotlin
abstract fun get(d: Double): (Int) -> String
// …
val getImpl: (Double) -> ((Int) -> String) = TODO()
// invoke即“调用”的意思
val s: String = getImpl(12.0).invoke(2)
// 等价写法
val s2: String = getImpl(12.0)(2)
```

## Lambda兼容SAM

Java中即便存在作用上相同的SAM，也不能混用，因此就造成可能要定义大量SAM。

```java
// Java
public interface SAM1 {
    String action(int i);
}

public interface SAM2 {
    String action(int i);
}

class Main {
    public static String runJava(SAM1 sam) {
        return sam.action(1);
    }

    public static void main(String[] args) {
        SAM1 sam1 = Integer::toString;
        SAM2 sam2 = Integer::toString;
        runJava(sam1);
        runJava(sam2); // ERROR
        runJava(Integer::toString);
    }
}
```

Kotlin显然没有这种问题，因为Kotlin中的函数类型就是以函数的结构定义的：

```kotlin
// Kotlin
fun runKt(block: (Int) -> String) = block.invoke(1)
// …
runKt(Int::toString)
```

## DSL Style

1. 当参数列表中最后一个参数为Lambda时，可以将大括号提出小括号：

   ```kotlin
   runKt { i ->
       return i.toString()
   }
   ```

2. Lambda可以省略 `return`，执行的最后一行的返回值作为Lambda的返回值：

   ```kotlin
   runKt { i ->
       if (i > 3) i - 3
       else i
   }
   ```

3. 如果Lambda的参数只有一个，不需要显式声明，默认为用`it`表示：

   ```kotlin
   runKt {
       if (it > 3) it - 3
       else it
   }
   ```

4. 对于多个参数，则必须显示声明，用不到的参数，用 `_` 表示：

   ```kotlin
   fun doWork(i1: Int, i2: Int, block: (Int, Int) -> Int): Int {
       return block(i1, i2)
   }
   // …
   doWork(1, 2) { _, i2 -> abs(i2) }
   ```

5. Lambda 内参数也可以显式其类型：

   ```kotlin
   doWork(1, 2) { i1: Int, i2: Int ->
        maxOf(i1, i2)
   }
   ```

   
