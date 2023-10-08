---
title: Compose 新手容易忽视的点（一）
date: 2023-07-18 23:49:02
tags: [kotlin, compose]
---

1. 使用 `remember` 避免在重组时反复获取对象：

   我们在 Composable函数体内，根据 UIState 或者 Composable参数 提供的值，动态的计算一些展示给用户的数据，
   可能需要进行数据的转换，其中可能涉及复杂的逻辑运算或者IO运算，重组时会导致其反复执行获取

   ```kotlin
   @Composable
   fun MainScreen(
       id: Int,
       modifier: Modifier = Modifier
   ) {
       // 'getById' is not a composable function, 
       // and it is always called after each recomposition
       val user = Users.getById(id)
       UserComponent(user, modifier)
       
       // ...
   }
   ```

   使用 `remember` 可以很好的避免重组过程中反复执行获取对象，他的 `block : () -> R` 只有当任意一个 key 与重组前对应值不等时重新构造

   ```kotlin
   @Composable
   fun MainScreen(
       id: Int,
       modifier: Modifier = Modifier
   ) {
       // recall 'getById' when id is changed
       val user = remember(id) {
           Users.getById(id)
       }
       UserComponent(user, modifier)
  
       // ...
   }
   ```

   如果你始终不想对象被重新获取，可以不传入 key

   ```kotlin
   @Composable
   fun MainScreen(
       id: Int,
       modifier: Modifier = Modifier
   ) {
       // never changed
       val user = remember {
           // logger.i("I will be invoked only once!")
           Users.getById(id)
       }
       UserComponent(user, modifier)
  
       // ...
   }
   ```

2. 使用 `CompositionLocal` 隐式传递参数：

   有一些参数不必作为 Composable 参数层层传递，例如 `Density`，他承载着屏幕像素密度的相关信息

   通过 Composable 参数层层传递的话，和 `Modifier` 类似，不过 `Density` 不同于 `Modifier`
   ，它是一个不需要在层级间转换的对象，所以将其做成一种静态获取的方式似乎更加合适

   Compose 为了解决这一问题，设计出了**隐式传参**， 你可以在 Composable 中使用 `LocalDensity.current` 获取 `Density` 对象，

   注意，这里的 `current` 的 getter 是一个 Composable 函数，因此不需要 `remember`

   ```kotlin
   @Composable
   fun MainScreen(
        id: Int,
        modifier: Modifier = Modifier
   ) {
        val density = LocalDensity.current
        
        // ...
   }
   ```

   类似的，我们也可以自定义隐式传参

   首先，定义想要被传参的类

   ```kotlin
   data class Theme(
        val primary: Color,
        val background: Color
   )
   ```

   然后，定义用于获取参数的 `CompositionLocal`， 它分为
   `ProvidableCompositionLocal`,`DynamicProvidableCompositionLocal`,
   以及 `StaticProvidableCompositionLocal` 三个子类型，其中`ProvidableCompositionLocal` 是其他两个的父类

   可以使用 `compositionLocalOf` 构造 `DynamicProvidableCompositionLocal`

   ```kotlin
   val LocalTheme = compositionLocalOf<Theme> {                                                       
        // if there is no instance found, use this default value
        defaultTheme
        // or throw an error
        // error("no theme provided")
   }
   ```

   而 `StaticProvidableCompositionLocal` 一般用于传递静态或者很少发生改变的对象，可以使用 `staticCompositionLocalOf` 构造

   ```kotlin
   // use StaticProvidableCompositionLocal is not recommend here
   val LocalTheme = staticCompositionLocalOf<Theme> {              
        error("no theme provided")
   }
   ```

   > 与 `compositionLocalOf` 不同，`staticCompositionLocalOf` 的读取不会被 composer 跟踪，
   并且在 `CompositionLocalProvider` 调用中更改提供的值将导致重新组合整个内容，而不仅仅是在组合中使用本地值的地方。

   最后，在你想的 Composable 位置，使用 `CompositionLocalProvider` 传入你想传递的对象

   ```kotlin
   @Composable
   fun App(
        modifier: Modifier = Modifier,
        content: @Composable () -> Unit
   ) {
        // It will be the default instance if parent composables do not provide as well
        val outerTheme = LocalTheme.current
        CompositionLocalProvider(
             LocalTheme provide lightTheme,
             // etc
        ) {
            // It will be lightTheme
            val innerTheme = LocalTheme.current
   
            // child composables could get local theme,
            // and they can also change current local theme in their scopes
            content()
   
            // ...
        }
   
        // ...
   }
   ```

3. 避免错误使用 `Modifier#onGloballyPositioned`:

   有时候我们需要测量组件的尺寸以及偏移量信息，可以使用 `Modifier#onGloballyPositioned`

   ```kotlin
   @Composable
   fun MainScreen(
       modifier: Modifier = Modifier
   ) {
       Column(modifier) {
           var height by remember { mutableStateOf(0) }
           var offset by remember { mutableStateOf(Offset.Zero) }
           Component(
               modifier = Modifier.onGloballyPositioned { layoutCoordinates ->
                    height = layoutCoordinates.size.height
                    offset = layoutCoordinates.positionInRoot()
               }
           )
           Text("height: $height, offset: $offset")
       }
   }
   ```

   当LayoutCoordinates可用时，此回调将至少被调用一次，并且每当元素在窗口内的位置发生变化时都会被调用。
   然而，并不能保证每次与修改的元素相对于屏幕的位置发生变化时都会调用该回调。例如，系统可能会在不触发回调的情况下移动窗口内的内容。
   如果您正在使用LayoutCoordinates来计算屏幕上的位置，而不仅仅是窗口内部的位置，则可能不会收到回调。

   如需测量尺寸，改为使用 `BoxWithConstraints`

   ```kotlin
   @Composable
   fun MainScreen(
       modifier: Modifier = Modifier
   ) {
       Column(modifier) {
           var height by remember { mutableStateOf(0) }
           BoxWithConstraints {
               height = constraints.maxHeight
               Componment()
           }
           Text("height: $height")
       }
   }
   ```
