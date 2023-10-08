## StateFlow, SharedFlow 和 ChannelFlow

1. 构造方式
    - StateFlow：
    ```kotlin
    // 构造 StateFlow 需要一个 value: T 作为初始值
    fun <T> MutableStateFlow(value: T): MutableStateFlow<T>
    // 例如
    fun createFlow(): MutableStateFlow<Int> = MutableStateFlow(0)
    ```
   - SharedFlow:
   ```kotlin
   // 构造 SharedFlow 不需要初始值，并且你可以指定一些详细的参数，后文会详细介绍
   fun <T> MutableSharedFlow(
       replay: Int = 0,
       extraBufferCapacity: Int = 0,
       onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND,
   ): MutableSharedFlow<T>
   // 例如
   fun createFlow(): MutableSharedFlow<Int> = MutableSharedFlow()
   ```
   - ChannelFlow:
   ```kotlin
   // ChannelFlow 属于冷流，使用 channelFlow builder 构造
   fun <T> channelFlow(@BuilderInference block: suspend ProducerScope<T>.() -> Unit): Flow<T>
   // 例如
   fun createFlow(): Flow<Int> = channelFlow { 
       // TODO()
   }
   ```
2. 不同场景不同表现
   ```kotlin
   val scope: CoroutineScope = CoroutineScope(Dispatchers.IO + Job())
   val flow = createFlow() 
   fun collect(): Job = scope.launch {     
       flow.collect {
           println(it)
       }
   }
   fun emit(i: Int = 0) {
       scope.launch {
           // can only be MutableStateFlow or MutableSharedFlow
           // because cold flow cannot emit out of flow builder
           flow.emit(i) 
       }
   }
   fun cancel(job: Job?) {
       job?.cancel() 
   }
   ```
   - 先 collect 后 emit
   ```kotlin
   collect()
   emit()
   // [StateFlow, SharedFlow, ChannelFlow]
   // 0
   // 三种 flow 表现一致，都能正常接收
   ```
   - 先 emit 后 collect
   ```kotlin
   emit()
   collect()
   // [StateFlow]
   // 0, <will collect stored value>
   // [SharedFlow]
   // <cannot collect stored value>
   // [ChannelFlow]
   // <cannot emit>
   // StateFlow 始终会接收最新的值，即便该值在 collect 之前就存在
   // SharedFlow 则不会接收到
   // ChannelFlow 更不会接收到因为没办法在 channelFlow {} 之外 emit
   ```
   - 先 collect 再 emit(0) 后 emit(1)
   ```kotlin
   collect()
   emit(0)
   emit(1)
   // [StateFlow, SharedFlow, ChannelFlow]
   // 0
   // 1
   // 三种 flow 表现一致，都能先后接收不同的值
   ```
   - 先 collect 后 emit(0) 两次
   ```kotlin
   collect()
   emit(0)
   emit(0)
   // [StateFlow]
   // 0
   // <will not collect repeated value>
   // [SharedFlow, ChannelFlow]
   // 0
   // 0
   // StateFlow 会根据 equals 自动过滤与上一次重复的值
   ```
   
> 未完结