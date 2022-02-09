### WorkManager特点：

- **保证用户的任务一定会执行**（记录更新每一个任务的信息/状态【Room数据库的更新】，手机重启，APP被杀掉 都一定会执行，因为同学们要记住一句话：Google说WorkManager是保证你的任务一定会执行的）

- **非及时性的执行任务**（就是不会马上执行，哪怕是你看到的现象是马上执行的，但每次执行的时间都无法确定，因为都是非及时性的执行任务哦）

### 适用场景：

定期重复性任务，但时效性/及时性要求不高的，例如：定期log上传，数据备份，等等

<img src="/Users/liujian/Documents/study/books/AndroidStudy/图片/image-20220201105132525.png" alt="image-20220201105132525" style="zoom:50%;" />

### 使用：

```kotlin
/*******先定义任务Worker，然后WorkManager.getInstance(this).enqueue(WorkRequest)，放入执行队列**/

// 最简单的 执行任务
class MainWorker1(context: Context, workerParameters: WorkerParameters) : Worker(context, workerParameters) {
    companion object { const val TAG = "Derry" }
    // 后台任务 并且 异步的 （原理：线程池执行Runnable）
    override fun doWork(): Result {
        Log.d(TAG, "MainWorker1 doWork: run started ... ")

        try {
            Thread.sleep(8000) // 睡眠
        } catch (e: InterruptedException) {
            e.printStackTrace()
            Result.failure(); // 本次任务失败
        } finally {
            Log.d(TAG, "MainWorker1 doWork: run end ... ")
        }
        return Result.success(); // 本次任务成功
    }
}

/**
 * 数据 互相传递
 * 后台任务
 */
class MainWorker2(context: Context, private val workerParams: WorkerParameters) : Worker(context, workerParams) {

    companion object { const val TAG = "Derry" }

    // 后台任务 并且 异步的 （原理：线程池执行Runnable）
    @SuppressLint("RestrictedApi")
    override fun doWork(): Result { // 开始执行了 ENQUEUED
        Log.d(MainWorker2.TAG, "MainWorker2 doWork: 后台任务执行了")

        // 接收 MainActivity传递过来的数据
        val dataString = workerParams.inputData.getString("Derry")
        Log.d(MainWorker2.TAG, "MainWorker2 doWork: 接收MainActivity传递过来的数据:$dataString")

        // 正在执行中 RUNNING

        // 反馈数据 给 MainActivity
        // 把任务中的数据回传到MainActivity中
        val outputData = Data.Builder().putString("Derry", "三分归元气").build()
        // return new Result.Failure(); // 本地执行 doWork 任务时 失败
        // return new Result.Retry(); // 本地执行 doWork 任务时 重试一次
        // return new Result.Success(); // 本地执行 doWork 任务时 成功 执行任务完毕
        return Result.Success(outputData) // if (workInfo.state.isFinished) Success
    }
}

/************放入队列—********/

//最简单的 执行任务 测试后台任务 1
fun testBackgroundWork1(view: View) {
    // OneTimeWorkRequest  单个 一次的
    val oneTimeWorkRequest = OneTimeWorkRequest.Builder(MainWorker7::class.java).build()
    WorkManager.getInstance(this).enqueue(oneTimeWorkRequest)
}

//数据 互相传递  测试后台任务 2
fun testBackgroundWork2(view: View?) {
    // 单一的任务  一次
    val oneTimeWorkRequest1: OneTimeWorkRequest
    // 数据
    val sendData = Data.Builder().putString("Derry", "九阳神功").build()
    // 请求对象初始化
    oneTimeWorkRequest1 = OneTimeWorkRequest.Builder(MainWorker2::class.java)
    .setInputData(sendData) // 数据的携带  发送  一般都是 携带到Request里面去 发送数据给 WorkManager2
    .build()
    // 一般都是通过 状态机 接收 WorkManager2的回馈数据
    // 状态机（LiveData） 才能接收 WorkManager回馈的数据
    WorkManager.getInstance(this).getWorkInfoByIdLiveData(oneTimeWorkRequest1.id)
    .observe(this, { workInfo ->
                    // ENQUEUED,RUNNING,SUCCEEED
                    Log.d(MainWorker2.TAG, "状态：" + workInfo.state.name)
                    // ENQUEUED, RUNNING  都取不到 回馈的数据 都是 null
                    // Log.d(MainWorker2.TAG, "取到了任务回传的数据: " + workInfo.outputData.getString("Derry"))
                    if (workInfo.state.isFinished) { // 判断成功 SUCCEEDED状态
                      Log.d(MainWorker2.TAG, "取到了任务回传的数据: " + workInfo.outputData.getString("Derry"))
                    }
                   })
    WorkManager.getInstance(this).enqueue(oneTimeWorkRequest1)
}

//多个任务 顺序执行 测试后台任务 3
fun testBackgroundWork3(view: View) {
    // 单一的任务  一次
    val oneTimeWorkRequest3 = OneTimeWorkRequest.Builder(MainWorker3::class.java).build()
    val oneTimeWorkRequest4 = OneTimeWorkRequest.Builder(MainWorker4::class.java).build()
    val oneTimeWorkRequest5 = OneTimeWorkRequest.Builder(MainWorker5::class.java).build()
    val oneTimeWorkRequest6 = OneTimeWorkRequest.Builder(MainWorker6::class.java).build()

    // 顺序执行 3  4  5  6
    WorkManager.getInstance(this)
    .beginWith(oneTimeWorkRequest3) // 做初始化检查的任务 成功后
    .then(oneTimeWorkRequest4) // 业务1 任务 成功后
    .then(oneTimeWorkRequest5)  // 业务2 任务 成功后
    .then(oneTimeWorkRequest6) // 最后检查工作任务
    .enqueue()

    // 需求：先执行  3  4    最后执行 6
    val oneTimeWorkRequests: MutableList<OneTimeWorkRequest> = ArrayList() // 集合方式
    oneTimeWorkRequests.add(oneTimeWorkRequest3) // 先同步日志信息
    oneTimeWorkRequests.add(oneTimeWorkRequest4) // 先更新服务器数据信息

    WorkManager.getInstance(this).beginWith(oneTimeWorkRequests)
    .then(oneTimeWorkRequest6) // 最后再 检查同步
    .enqueue()
}

//重复执行后台任务  非单个任务，多个任务  测试后台任务 4
fun testBackgroundWork4(view: View) {
        // OneTimeWorkRequest 单个  前面的三个例子  不会轮询 执行一次就OK

        // 重复的任务  多次/循环/轮询  , 哪怕设置为 10秒 轮询一次,   那么最少轮询/循环一次 15分钟（Google规定的）
        // 不能小于15分钟，否则默认修改成15分钟
        val periodicWorkRequest =
            PeriodicWorkRequest.Builder(MainWorker3::class.java, 10, TimeUnit.SECONDS)
            .build()

        // 【状态机】  为什么一直都是 ENQUEUE，因为 你是轮询的任务，所以你看不到 SUCCESS        [如果你是单个任务，就会看到SUCCESS结束任务]
        // 监听状态
        WorkManager.getInstance(this).getWorkInfoByIdLiveData(periodicWorkRequest.id)
            .observe(this, { workInfo ->
                Log.d(MainWorker2.TAG, "状态：" + workInfo.state.name) // ENQUEEN   RUNN  循环反复
                if (workInfo.state.isFinished) {
                    Log.d(MainWorker2.TAG, "状态：isFinished=true 同学们注意：后台任务已经完成了...")
                }
            })
        WorkManager.getInstance(this).enqueue(periodicWorkRequest)

        // 取消 任务的执行
        // WorkManager.getInstance(this).cancelWorkById(periodicWorkRequest.getId());
}

//约束条件，约束后台任务执行 测试后台任务 5 
@RequiresApi(api = Build.VERSION_CODES.M)
fun testBackgroundWork5(view: View?) {
    val constraints = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.CONNECTED) // 必须是联网中
    /*.setRequiresCharging(true) // 必须是充电中
              .setRequiresDeviceIdle(true) // 必须是空闲时（例如：你没有玩游戏  例如：你有没有看大片 1亿像素的）*/
    .build()

    /**
           * 除了上面设置的约束外，WorkManger还提供了以下的约束作为Work执行的条件：
           * setRequiredNetworkType：网络连接设置
           * setRequiresBatteryNotLow：是否为低电量时运行 默认false
           * setRequiresCharging：是否要插入设备（接入电源），默认false
           * setRequiresDeviceIdle：设备是否为空闲，默认false
           * setRequiresStorageNotLow：设备可用存储是否不低于临界阈值
           */
    // 请求对象
    val request = OneTimeWorkRequest.Builder(MainWorker3::class.java)
    .setConstraints(constraints) // Request 关联  约束条件
    .build()

    // 加入队列
    WorkManager.getInstance(this).enqueue(request)
}

```

### 原理

首先**WorkManager****的初始化**不实在WorkManager.getInstance(this)，而是apk在打包的时候，在Manifest文件中会自动加入provider的注入

<img src="/Users/liujian/Documents/study/books/AndroidStudy/图片/image-20220201110710294.png" alt="image-20220201110710294" style="zoom:50%;" />

根据provider的加载时间，会在应用启动的时候，自动加载，然后完成第一次初始化。

```java
WorkManagerInitializer： 
public class WorkManagerInitializer extends ContentProvider {
  @Override public boolean onCreate() { 
    // Initialize WorkManager with the default configuration. 
    WorkManager.initialize(getContext(), new Configuration.Builder().build()); 
    return true; 
  } 
}
```

① WorkManager的初始化是由WorkManagerInitializer这个ContentProvider

执行的，同学们可以看看

② 会初始化 Configuration，WorkManagerTaskExecutor，WorkDatabase，

Schedulers，Processor

③ GreedyScheduler 埋下伏笔

④ 发现了未完成的，需要重新执行的任务（之前 意外 中断 的继续执行）



同时会在在Manifest文件中注入约束条件广播的监听，比如电量、网络等。

执行原理：work任务是启动系统进程将任务写入room，通过轮询机更新状态，这样不会丢失任务，启动的时候去查找数据库，查看任务执行状态，同步任务。

<img src="/Users/liujian/Documents/study/books/AndroidStudy/图片/image-20220201111353420.png" alt="image-20220201111353420" style="zoom:50%;" />