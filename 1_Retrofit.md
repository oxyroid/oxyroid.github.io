## Retrofit

### 概览

- ‎Retrofit‎‎ 是将 API 接口转换为可调用对象的工具。‎

- 本章节用到的全部依赖如下
  - [ retrofit2 : retrofit ](https://mvnrepository.com/artifact/com.squareup.retrofit2/retrofit)
  - [ retrofit2 : converter-gson](https://mvnrepository.com/artifact/com.squareup.retrofit2/converter-gson)

### 使用 Retrofit 发送简单的 HTTP 请求

* 现有一个后端接口 `localhost:8080/test/currentTime`，请求后会返回当前时间（毫秒时，Long型）。

* 我们在项目根下新建`Contracts`单例类用于保存全局常量，并将公共请求前缀写入：

  ```kotlin
  // Contracts
  object Contracts {
      // 不要忘记在最后补上一个斜杠
      const val BASE_URL = "https://$LOCAL_HOST:$PORT/test/"
      private const val LOCAL_HOST = "192.168.18.3"
      private const val PORT = "8080"
  }
  ```
  
* 在项目根目录下新建包`data.api`用于存放`Retrofit`接口，然后在该包下新建`TestApi`接口：

  ```kotlin
  // data.api.TestApi
  interface TestApi {
      // 在这里定义你的第一个请求
      @GET("currentTime")
      suspend fun currentTime(): Call<Long>
  }
  ```

  > `@GET` 表明该请求是个GET请求，它接收参数 `value:String` 表示具体的请求地址

* 然后我们可以在当前接口的伴生类中使用 `Retrofit.Builder` 实例化一个默认的对象：

  ```kotlin
  // data.api.TestApi
  interface TestApi {
      ...
      companion object {
          val Default = Retrofit.Builder()
          	.baseUrl(Contracts.BASE_URL)
          	.build()
          	.create(TestApi::class.java)
          	//或者 ".create<TestApi>()"
      }
  }
  ```

* 现在我们在来测试这个接口：

  ```kotlin
  // ui.TestActivity
  class TestActivity : AppCompatActivity() {
      private val testApi = TestApi.Default
      private lateinit var button: Button
      private lateinit var textView: TextView
      override fun onCreate(savedInstanceState: Bundle?) {
          ...
          button = findViewById<Button>(R.id.button)
          textView = findViewById<TextView>(R.id.textView)
          button.setOnClickListener {
              lifecycleScope.launch(Dispatcher.IO) {
              	val call = testApi.currentTime()
              	call.enqueue(object : Callback<Long> {
                  	override fun onResponse(
                      	call: Call<Long>,
                      	response: Response<Long>
                  	) {
                      	textView.text = "${response.body()}ms"
                  	}
                  	override fun onFailure(call: Call<Long>, t: Throwable) {
                      	Log.e(TAG, "onCreate", t)
                  	}
              	})
          	}
          }
      }
  }
  ```

### 处理 Retrofit 接口返回的 JSON 格式的请求结果

- 现有一个后端接口 `localhost:8080/test/user/random` ，请求后会已JSON格式返回随机用户简略信息，比如：

  ```json
  {
      "userId": "40019",
      "name": "李亮",
      "lastOnlineAt": 1651620000000
  }
  ```

- 我们先定义一个`UserDTO`数据类：

  ```kotlin
  // data.dto.UserDTO
  data class UserDTO (
  	@SerializedName("userId") val id : Int,
      val name : String,
      val lastOnlineAt : Long
  )
  ```

  > 我们不需要将JSON的全部内容定义为字段，按需即可

- 在`UserApi`中定义该请求：

  ```kotlin
  //data.api.UserApi
  interface UserApi {
      @GET("user/random")
      suspend fun getRandomUser(): User
      companion object {
          val Default = Retrofit.Builder()
          	.baseUrl(Contracts.BASE_URL)
          	// 使用Gson转换器
              .addConverterFactory(GsonConverterFactory.create())
          	.build()
          	.create(UserApi::class.java)
      }
  }
  ```

- 后端返回的`lastOnlineAt`是我们当前不需要的，我们可以定义一个 `User` 数据类，然后为 `UserDTO` 定义一个扩展方法`toUser(): User` ：

  ```kotlin
  // domain.entity.User
  data class User (
  	val id: Int,
      val name: String
  )
  ```

  ```kotlin
  // data.dto.UserDTO
  data class UserDTO (...)
  
  fun UserDTO.toUser(): User = User(id, name)
  ```

- 测试这个接口：

  ```kotlin
  // ui.TestActivity
  class TestActivity: AppCompatActivity(){
      ...
      override fun onCreate(savedInstanceState: Bundle?) {
          ...
          lifecycleScope.launch(Dispatcher.IO) {
              val user = try {
                  userApi.getRandomUser().toUser()
              } catch (e: Exception){
                  Log.e(TAG, "onCreate:", e)
                  null
              }
              withContext(Dispatcher.Main) {
                  showUser(user)
              }
          }
      }
      
  	@MainThread
      private fun deliverUser(user: User?) {
          textView.text = user?.name?: "Network Error."
      }
  }
  ```


### 使用 @Path 和 @Query 传入参数

- 现有一个API接口 `localhost:8080/test/dog/{count}`，`count` 传入一个整数，后端会以GSON数组的格式随机返回狗狗的简略信息，例如：

  ```json
  // localhost:8080/test/dog/2
  [
      {
          "name": "Kitty",
          "age": 2,
          "sex": "female"
      },
      {
          "name": "Lucio",
          "age": 5,
          "sex": "male"
      }
  ]
  ```

- 定义数据类 `DogDTO`

  ```kotlin
  // data.dto.DogDTO
  data class DogDTO (
  	val name: String,
      val age: Int,
      val sex: String
  )
  ```

- 定义接口 `DogApi`，通过 `@Path` 注解实现动态请求：

  ```kotlin
  // data.api.DogApi
  interface DogApi {
      @GET("dog/{count}")
      suspend fun getDogs(@Path("count") count: Int = 1): List<DogDTO>
  }
  ```

- 如果API接口需要传入的参数包含**键值对**格式，如 `localhost:8080/test/dog/{count}?age={age}&sex={sex}`，那么就需要用到`@Query`注解：

  ```kotlin
  // data.api.DogApi
  interface DogApi {
      @GET("dog/{count}")
      suspend fun getDogs(
          @Path("count") count: Int = 1,
          @Query("age") age: Int,
          @Query("sex") sex: String
      ): List<DogDTO>
  }
  ```

   



