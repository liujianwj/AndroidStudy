- service 的生命周期，两种启动方式的区别

  > - 启动服务
  >
  >   startService执行的生命周期：onCreate()->onStartCommand()->onDestory()
  >
  >   1）启动服务/startService() 
  >
  >         单次启动：onCreate()->onStartCommand()
  >     
  >         多次启动：onCreate()->onStartCommand()->onStartCommand()...
  >     
  >         注意：多次启动onCreate() 只会调用一次
  >
  >   2）停止服务/stopService()
  >
  >        onDestory()
  >
  > - 绑定服务
  >
  >   bindService执行的生命周期：onCreate()->onBind()->unbindService()->onDestory()
  >
  >   1）绑定服务/bindService()
  >
  >         onCreate()->onBind()
  >
  >   2）解绑服务/unbindService()
  >
  > 启动服务：不会随调用者摧毁而摧毁
  >
  > 绑定服务：可以与调用者绑定实现一些交互，但是与调用者共生死

- 如何保证Service不被杀死 ？

  > 没办法优雅的处理，优先级？自启动？alertmanager?

- Service与Activity怎么实现通信

  >bindService，通过serviceConnection获取ibinder交互

- IntentService是什么,IntentService原理，应用场景及其与Service的区别

  > 应用场景启动app后，后台自动下载资源，不用管生死，完成任务自己销毁。

- Service 的 onStartCommand 方法有几种返回值?各代表什么意思?

  > 1、START_STICKY： 如果service进程被kill掉，保留service的状态为开始状态，但不保留递送的intent对象。随后系统会尝试重新创建service，由 于服务状态为开始状态，所以创建服务后一定会调用onStartCommand(Intent,int,int)方法。如果在此期间没有任何启动命令被传 递到service，那么参数Intent将为null。
  >
  > 2、START_NOT_STICKY：“非粘性的”。使用这个返回值时，如果在执行完onStartCommand后，服务被异常kill掉，系统不会自动重启该服务
  >
  > 3、START_REDELIVER_INTENT：重传Intent。使用这个返回值时，如果在执行完onStartCommand后，服务被异常kill掉，系统会自动重启该服务，并将Intent的值传入。
  >
  > 4、START_STICKY_COMPATIBILITY：START_STICKY的兼容版本，但不保证服务被kill后一定能重启。

- bindService和startService混合使用的生命周期以及怎么关闭

  > 混合使用可以结合两者优点，既能与调用者实现一些交互，又不随调用者共生死
  >
  > onCreate – onStartCommand – onBind – onUnbind–onDestroy
  >
  > 关闭：先unbindService，然后stopService

- 用过哪些系统Service ？

  > wms、pkms、alaretmanager

- 了解ActivityManagerService吗？发挥什么作用

  > AMS，四大组件的管理