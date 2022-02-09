1. service 的生命周期，两种启动方式的区别

   https://www.jianshu.com/p/cc25fbb5c0b3

   - 启动服务

     startService执行的生命周期：onCreate()->onStartCommand()->onDestory()

     1）启动服务/startService() 

     ​      单次启动：onCreate()->onStartCommand()

     ​      多次启动：onCreate()->onStartCommand()->onStartCommand()...

     ​      注意：多次启动onCreate() 只会调用一次

     2）停止服务/stopService()

     ​     onDestory()

   - 绑定服务

     bindService执行的生命周期：onCreate()->onBind()->unbindService()->onDestory()

     1）绑定服务/bindService()

     ​      onCreate()->onBind()

     2）解绑服务/unbindService()

   启动服务：不会随调用者摧毁而摧毁

   绑定服务：可以与调用者绑定实现一些交互，但是与调用者共生死

2. Service启动流程

3. Service与Activity怎么实现通信

   bindService，通过serviceConnection获取ibinder交互

4. IntentService是什么,IntentService原理，应用场景及其与Service的区别

   应用场景启动app后，后台自动下载资源，不用管生死，完成任务自己销毁。

5. Service 的 onStartCommand 方法有几种返回值?各代表什么意思?

6. bindService和startService混合使用的生命周期以及怎么关闭

   混合使用可以结合两者优点，既能与调用者实现一些交互，又不随调用者共生死

   onCreate – onStartCommand – onBind – onUnbind–onDestroy

   关闭：先unbindService，然后stopService

