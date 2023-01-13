## 初探 Reactive Programming

### 概览

- 本章将介绍响应式编程并讲解 **LiveData** 和 **Flow** 组件的原理以及使用

### 生产者 - 消费者

- 通常负责从本地数据库或远程数据库获取数据的组件我们称之为**生产者**，而使用数据并进行“展示”等操作的组件我们称之为**消费者**
- 在传统的架构模式下，我们总是习惯于在我们需要的时候通知生产者进行**生产**数据，利用接口回调的方式，处理数据然后将结果用于消费者的**消费**行为
- 不过，这种传统的“生产-消费”模式往往会将**数据操作**、**逻辑操作**以及**UI操作**耦合，当项目变得稍微庞大一些，或者想要修改陈旧代码的时候往往会变得十分困难，甚至在不同数据需要互相组合调用的时候更容易写出BUG

### 什么是响应式编程

- 响应式编程巧妙地解决了上面的问题，我们通过在 生产者 - 消费者 当中添加一个“中介”负责管理数据的存储与分发，我们称这个中介为 **可观察数据** 或 **响应式数据**
- 生产者生产的数据可以存放到这个可观察数据中，消费者可以通过**观察者**订阅这个可观察数据，生产者/消费者 与可观察数据的关系是多对多的，每当因生产者生产行为导致可观察数据的值改变，可观察数据（通常情况下）会立即通知所有订阅他的观察者以便消费者进行消费

### LifecycleOwner

- LifecycleOwner是一个接口，通过 `LifecycleOwner#getLifecycle(): Lifecycle` ，可以获取到Lifecycle对象
- Lifecycle是定义一个具有 Android 生命周期的对象，通过`Lifecycle#getCurrentState(): State`，可以获取到当前生命周期的状态，如 **ON_CREATE**，**ON_START** 等

### Observer

- Observer是一个 **SAM接口** ，是可以从 LiveData 接收数据的简单回调 `Observer#onChange(T)` 

### LiveData

- LiveData 是一个可以在给定生命周期内观察到的数据持有者类。这意味着Observer可以与LifecycleOwner成对添加，并且只有当配对的 LifecycleOwner 处于活动状态时，才会通知该Observer数据发生的变化

- 在Android中，**ComponentActivity** 以及 **Fragment** 都已默认实现了LifecycleOwner接口

    ```kotlin
    // data.source.MessageDataSource (生产者)
    class MessageDataSource {
        // ...
        fun getMessage(): String {
            return mockApi.getMessage()
        }
    }
    ```

    ```kotlin
    // ui.MessageViewModel (存放可观察数据)
    class MessageViewModel(
    	private val dataSource: MessageDataSource
    ): ViewModel() {
        // MutableLiveData是LiveData的子类，其拥有改变其持有数据的APIs
        val messageLiveData = MutableLiveData<String>()
        fun getMessge() {
            // 同步方式改变其持有数据：MutableLiveData<T>#setValue(T)
            // 异步方式改变其持有数据：MutableLiveData<T>#postValue(T)
            messageLiveData.value = dataSource.getMessage()
        }
    }
    ```

    ```kotlin
    // ui.MessageActivity (消费者)
    // AppCompatActivity是ComponentActivity的子类
    class MessageActivity: AppCompatActivity() {
        // ComponentActivity#viewModels((() -> Factory)): Lazy<VM>
        // 返回一个 Lazy 委托以访问 ComponentActivity 的 ViewModel
        private val viewModel by viewModels<MessageViewModel>()
        private lateinit var textView: TextView
        private lateinit var button: Button
        override fun onCreate(savedInstanceState: Bundle?) {
            // 订阅此数据：LiveData#observe(LifecycleOwener, Observer)
            viewModel.messageLiveData.observe(this) {
                textView.text = it
            }
            button.setOnClickListener {
                // 触发数据获取操作
                viewModel.getMessage()
            }
        }
    }
    ```
    

### MediatorLiveData

- MediatorLiveData是MutableLiveData子类，它可以观察其他LiveData对象并对来自它们的OnChanged事件作出反应

    ```kotlin
    val livedata = MediatorLiveData<String>()
    // childLivedata1: LiveData<Int>
    livedata.addSource(childLivedata1) {
    	livedata.value = it.toString()
        println("The livedata1 has changed !")
    }
    // childLivedata2: LiveData<String>
    livedata.addSource(childLivedata2) {
    	livedata.value = "Hi, $it."
    }
    ```



### 私有化写入权限

- 通常我们不希望让MutableLiveData的setValue或postValue透露在ViewModel以外，我们可以定义一个字段将MutableLiveData **向上转型** 为LiveData，并仅允许外部通过该字段订阅：

    ```kotlin
    // viewmodel
    // 我们通常令私有的livedata命名为 “_ + 公有的livedata命名” 的格式
    private val _livedata = MutableLiveData<String>()
    val livedata: LiveData<String> get() = _livedata
    ```

    

### LiveData-ktx

LiveData-ktx 是官方为 LiveData 定义的一系列扩展方法，利用好它能提高开发效率

#### Flow.asLiveData

- 可以将Flow转换为LiveData

#### LiveData Builder

- 先看函数定义

    ```kotlin
    public fun <T> liveData(
        context: CoroutineContext = EmptyCoroutineContext,
        // DEFAULT_TIMEOUT = 5000L
        timeoutInMs: Long = DEFAULT_TIMEOUT,
        @BuilderInference block: suspend LiveDataScope<T>.() -> Unit
    ): LiveData<T>
    ```

    > `[suspend] emit<T>(T)` 用于在block中提交新值
    >
    > `[suspend] emitSource<T>(LiveData<T>): DisposableHandle` 用于在block中订阅其他LiveData
    >
    > ` latestValue: T?` 用于在block中获取最新提交的值

- 简单用例

    ```kotlin
    // 这里使用get即可实现延迟加载并且每次拿到的livedata都是不同实例
    // 如果仅想要延迟加载并只存在一个实例使用by lazy即可
    val livedata get() = livedata {
        // repeat(Int, (Int) -> Unit)
        // 执行给定的函数动作指定的次数。当前迭代的从零开始的索引作为参数传递给操作
        repeat(10) {
            emit(it)
            // [suspend] delay(Long)
            // 将协程延迟给定时间而不阻塞线程，并在指定时间后恢复它
            delay(1000)
        }
    }
    // LiveData#observeForever(Observer<T>)
    // 不需要提供LifecycleOwner即可订阅，同时也意味着只要数据发生改变回调就会执行
    livedata.observeForever(::println)
    ```

#### Map & SwitchMap



