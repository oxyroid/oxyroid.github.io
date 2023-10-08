## Kotlin Coroutine 挂起本质

 suspendCoroutine { con -> … } 会先生成一个SafeContinuation对象，然后将其传入block，即 “{ con -> … } ”。

换句话说这个block内部的参数con 实际上就是这个 传入的SafeContinuation对象，即在其中调用con.resumeWith本质上就是调用SafeContinuation.resumeWith。

最后suspendCoroutine会调用SafeContinuation对象的getOrThrow作为返回值返回给suspendCoroutineUninterceptedOrReturn，进而返回出去。

```kotlin
suspend inline <T> fun suspendCoroutine(
  crossinline block: (Continuation<T>) -> Unit
): T {
//…
// 注意，这个函数的 block 返回值是 Any?
	return suspendCoroutineUninterceptedOrReturn { c: Continuation<T> ->
      // 注意，这里调用的次构造函数
      val safe = SafeContinuation(c.intercepted())
  		block(safe)
  		safe.getOrThrow()
	}
}
```

接下来我们看一下SafeContinuation的大致结构：

```kotlin
internal actual class SafeContinuation<in T>
internal actual constructor(
    private val delegate: Continuation<T>,
    initialResult: Any?
) : Continuation<T>, CoroutineStackFrame {
    internal actual constructor(delegate: Continuation<T>) : this(delegate, UNDECIDED)

    @Volatile
    private var result: Any? = initialResult

    private companion object {
        @Suppress("UNCHECKED_CAST")
        private val RESULT = AtomicReferenceFieldUpdater.newUpdater<SafeContinuation<*>, Any?>(
            SafeContinuation::class.java, Any::class.java as Class<Any?>, "result"
        )
    }

    public actual override fun resumeWith(result: Result<T>) = ...

    internal actual fun getOrThrow(): Any? = ...
  
  	// ...
}
```

SafeContinuation伴生类里有一个状态机RESULT，负责对SafeContinuation实例的 result 成员进行原子更新，比如调用RESULT.compareAndSet(safe, UNDECIDED, result.value) 代表safe的result被从UNDECIDED更新成了result.value。

如果你在block执行完毕之前就调用了safe.resumeWith，那SafeContinuation 内部的调用顺序就是 resumeWith -> getOrThrow。如果你在执行block时把safe实例保存了下来，在block执行完毕后再调用safe.resumeWith，那SafeContinuation 内部的调用顺序就是 getOrThrow -> resumeWith。这两种方式就决定了函数是否真正的挂起：

```kotlin
public actual override fun resumeWith(result: Result<T>) {
		while (true) {
    		val cur = this.result
        when {
        		cur === UNDECIDED -> if (RESULT.compareAndSet(this, UNDECIDED, result.value)) return
            cur === COROUTINE_SUSPENDED -> if (RESULT.compareAndSet(this, COROUTINE_SUSPENDED, RESUMED)) {
            		delegate.resumeWith(result)
                return
            }
            else -> throw IllegalStateException("Already resumed")
        }
    }
}

internal actual fun getOrThrow(): Any? {
		var result = this.result
    if (result === UNDECIDED) {
    		if (RESULT.compareAndSet(this, UNDECIDED, COROUTINE_SUSPENDED)) return COROUTINE_SUSPENDED
        result = this.result
    }
    return when {
    		result === RESUMED -> COROUTINE_SUSPENDED
        result is Result.Failure -> throw result.exception
        else -> result
    }
}

```

提醒！suspendCoroutine内部创建的SafeContinuation的result初始值是*UNDECIDED*。

1. 先调用 resumeWith：

   result会先从UNDECIDED变成具体的值，因此等到getOrThrow调用时，就直接返回出去了，就和调用普通函数一样。

2. 后调用 resumeWith:

   result初始值UNDECIDED，getOrThrow调用时，result会被更新为COROUTINE_SUSPENDED，并且返回COROUTINE_SUSPENDED给外部。

总结，getOrThrow可能返回COROUTINE_SUSPENDED或者result: Any?（可能仍然是COROUTINE_SUSPENDED），而外部还有一层 suspendCoroutineUninterceptedOrReturn ，我们到看一下他的实现：

```kotlin
public suspend inline fun <T> suspendCoroutineUninterceptedOrReturn(crossinline block: (Continuation<T>) -> Any?): T {
    contract { callsInPlace(block, InvocationKind.EXACTLY_ONCE) }
    throw NotImplementedError("Implementation of suspendCoroutineUninterceptedOrReturn is intrinsic")
}
```

intrinsic 意味着 他的实现是在是内置于 compiler 的，不过我们阅读他的注释可以知道，他用于获取在挂起函数内部的当前 Continuation 实例，可以选择挂起当前运行的协程，或立即返回结果而不进行挂起：

1. 如果代码块返回特殊的 COROUTINE_SUSPENDED 值，这意味着挂起函数已挂起执行，不会立即返回任何结果。在这种情况下，**提供给代码块的 Continuation （上文的 c.intercepted(): Continuation）将在将来的某个时刻通过调用 resumeWith 来恢复以便继续计算**。
2. 否则，代码块的返回值必须具有可分配给 T 类型的类型，并表示此挂起函数的结果。这意味着执行未被挂起，并且提供给代码块的 Continuation 将不会被调用。

所以，挂起本质上是getOrThrow返回COROUTINE_SUSPENDED进而suspendCoroutineUninterceptedOrReturn阻塞了执行。

而恢复的本质是 resumeWith 的调用，result会被从COROUTINE_SUSPENDED更新为RESUMED，并且suspendCoroutineUninterceptedOrReturn的参数 c 的 c.intercepted() : Continuation 会被调用 resumeWith。

而一个未挂起过直接返回确切值的SafeContinuation，我们认为他“并未真正挂起”。

