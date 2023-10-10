---
title: 开发中的Tips（一）
date: 2023-10-09 15:40:20
tags: [ android, tips ]
---

### Room

#### Dao 定义的操作是否需要 `suspend`

总结就是：返回 `flow`, `livedata`等 的不需要，其他的推荐需要。

1. flow 只是一个数据流，协程只作用在 collect 时；
   ```kotlin
   interface TaskDao {
        @Query("SELECT * FROM tasks")
        fun observeAll(): List<Task>
   }
   dao
      .observeAll()
      .fileter { it.invalidate }
      .map { it.toUiModel() }
      .collect { deliverTask(it) } // <suspend>
   ```
2. 其他操作，加不加 suspend 都是同步调用，
    - 加就是挂起协程，不加就是阻塞线程。
    - 协程支不支持并行，挂起对子任务都不会有影响
    - 但是不支持并行（串行）的协程被阻塞线程，子任务也会被阻塞。
    - 加他会限制只能在协程作用域当中调用，不加可以在协程作用域里外调用。
    - 根据业务决定，如果需要在 Kotlin Coroutine 和 Java 中调用，推荐写两个：
   ```kotlin
   interface TaskDao { 
        @Query("SELECT * FROM tasks")
        suspend fun getAll(): List<Task>
        @Query("SELECT * FROM tasks")
        fun getAllBlocking(): List<Task> // <blocking> for Java
   }
   coroutineScope.launch {
        dao.getAll()
   }
   thread {
        dao.getAllBlocking()
   }
     ```
   > 我的建议还是直接操控 协程 而不是 线程