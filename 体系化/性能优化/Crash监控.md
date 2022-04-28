Java层的实现

- CPU的指标
- 内存的指标
- FPS的指标
- ANR
- 卡顿
- gc/oom
- 网络层面（http）hook
- 功耗
- 远程下发 日志回捞（push）支持远端shell动态代码下发





### 内存

1. 如果你想查看所有进程的内存使用情况，可以使用命令procrank、dumpsys meminfo查看，当然也只可以过滤出某个进程如：dumpsys meminfo | grep -i phone；

   uss：进程内存占用空间；

   使用procrank工具进行，android应用内存占用测试，写python脚本，每隔一秒打印procrank的信息 https://www.cnblogs.com/chengchengla1990/archive/2016/10/21/5984084.html；

   Adb shell top https://www.cnblogs.com/Qliupeng/p/9416219.html；

   android kill机制，Oom_adj，每一个进程都有一个adj，adj对应的数值越大越容易被kill；

   profile memory，锯齿状，查看内存的抖动；

2. jni泄漏，xhook c语言开辟内存的方法，看是否有关闭

3. 线上方案：registerActivitylifecycle，在onDestory时候，将activity添加进weekhashmap（key=activity实例，value是类名），然后根据onstart/onstop，判断应用是否在后台，在后台的时候通过，在onstop方法中去遍历weekhashmap，看有没有那个类在weekhashmap中存在的次数超过阈值，超过了则手动触发gc（1、分配大内存，2.gc，3.sleep 100）,然后再检查weekhashmap是否还是存在，存在的话则上报内存泄漏。

### 卡顿

1. systrace，Treace.beginSection，，，线上包需要反射
2. 线上：looper.setMessageLogging

### 布局优化

1. sub,merge,include,cliprect,bg,
2. 布局耗时监控，Factory2，在里面设置监听时间，对每个view的耗时进行统计，然后分析处理
3. 优化：1）异步加载，AsynclayoutInflater（缺点：xml不能有fragment）；2）x2c，编译时注解，直接new

 ### 网络优化

1.  Dns解析
2. 数据压缩
3. 连接优化

apm

- 配置（注解+json）

- 封装一个shell方法，可以通过adb命令获取各种信息，手机型号、屏幕、CPU信息、内存信息等

- 流量监控、FPS监控、网络请求耗时、方法耗时、anr、java层crash、内存泄漏（两次gc，一般不开放）

  

  

  

  

  