# 在实际开发中学习设计模式（一）

## 前言

今天在Code Review 项目时，遇到了一个需要翻转List的需求。

放在Java里，不借助第三方工具包的前提想要实现需求可能要反向遍历原列表，将元素一个个add到另外一个新的列表中。

```java
List<Object> list = getList();
ArrayList<Object> reversedList = new ArrayList<>();
for(int i = list.size() - 1; i >= 0; i--) {
    reversedList.add(list.get(i));
}
```

或者通过 `void Collections#reverse(List list)` 在原列表中翻转，不过这不是我们希望的，因为这样限制了传入的 `list` 必须是 **Mutable** 的，并且直接对参数修改不符合语义，容易造成 bug 且是非线程安全的。

```java
List list = getList();
Collections.reverse();
```

多亏Kotlin的 **Extension Function** 外加丰富的标准库函数，可以直接调用 `List.reversed()` 函数，返回一个翻转了的新实例，省掉了Java中唠叨的遍历。

```kotlin
val list = getList()
val reversedList = list.reversed()
```

下面是标准库中对 `List.reversed()` 函数的实现（实际上是对List的父接口 `Iterable` 的扩展实现）：

```kotlin
public fun <T> Iterable<T>.reversed(): List<T> {
    if (this is Collection && size <= 1) return toList()
    val list = toMutableList()
    // 这里是 Native 实现
    list.reverse()
    return list
}
```

> 明明 Kotlin 中 public 是默认修饰，是可以省略掉的，不懂为啥标准库都喜欢写上



## 委托模式

不过翻转这一操作本来 **O(n)** 复杂度就高，数据量一大，在实时刷新的 UI 中，必然会造成卡顿（其实可以通过把耗时任务提到其他 Layer 解决，不过这不是关键）。

后来，我发现除了 `List.reversed()` 函数，Kotlin标准库还定义了一个与之命名很像的函数—— `List.asReversed()`，敏锐的感觉这个 "as" 一定是用来解决 非"as" 的某些问题的。 

```kotlin
public fun <T> List<T>.asReversed(): List<T> = ReversedListReadOnly(this)
```

这个函数将自身的实例传递给了一个叫做 `ReversedListReadOnly` 的类。

> 注意：与Java不同的是，Kotlin 中的 List 仅具有Readable特性，如 get，size；不具备 Mutable（Writable） 特性，如 add，set，remove 等，这些 Mutable 特性被封装在了List的子接口 MutableList 中，这一设计在 Kotlin 中被称为“读写分离”

```kotlin
private open class ReversedListReadOnly<out T>(private val delegate: List<T>) : AbstractList<T>() {
    override val size: Int get() = delegate.size
    override fun get(index: Int): T = delegate[reverseElementIndex(index)]
}
```

打开源码发现， `ReversedListReadOnly` 和 `AbstractList` 都是`List` 的子类，显然 `ReversedListReadOnly` 是 `List` 的委托类，他重写了 `get(index: Int): T` 方法，使用 `reverseElementIndex(index: Int): Int` 返回的下标对被委托的原列表进行取值。不出意外的话，`reverseElementIndex` 的实现应该是返回翻转的下标：

```kotlin
private fun List<*>.reverseElementIndex(index: Int) =
    if (index in 0..lastIndex) lastIndex - index else throw IndexOutOfBoundsException("Element index $index must be in range [${0..lastIndex}].")
```

> 注意：这里的 `lastIndex: Int` 是 Kotlin 对 List 扩展的便捷属性，其值为 `size - 1`，代表列表末端的下标

```kotlin
public val <T> List<T>.lastIndex: Int
    get() = this.size - 1
```

这样就巧妙地**假装**翻转了列表。

## 结语

Kotlin 通过运用委托模式简单的包装一层容器，就将翻转列表这一需求**从复杂度 O(n) 降低为 O(1)**。这一源码级别的优秀范例让只是对着枯燥课本学习设计模式的我充分感悟到委托设计模式的有用之处。
