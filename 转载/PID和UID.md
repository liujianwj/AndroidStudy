## Android中PID与UID的作用与区别

原文链接：https://www.cnblogs.com/perseus/articles/2354173.html



**PID**: 为Process Identifier，　PID就是各进程的身份标识,程序一运行系统就会自动分配给进程一个独一无二的PID。进程中止后PID被系统回收，可能会被继续分配给新运行的程序，但是在android系统中一般不会把已经kill掉的进程ID重新分配给新的进程，新产生进程的进程号，一般比产生之前所有的进程号都要大。



**UID**:一般理解为User Identifier,UID在linux中就是用户的ID，表明时哪个用户运行了这个程序，主要用于权限的管理。而在android 中又有所不同，因为android为单用户系统，这时UID 便被赋予了新的使命，数据共享，为了实现数据共享，android为每个应用几乎都分配了不同的UID，不像传统的linux，每个用户相同就为之分配相同的UID。（当然这也就表明了一个问题，android只能时单用户系统，在设计之初就被他们的工程师给阉割了多用户），使之成了数据共享的工具。



因此在android中PID，和UID都是用来识别应用程序的身份的，但UID是为了不同的程序来使用共享的数据。

在android 中要通过UID共享数据只需在程序a,b中的menifest配置即可，具体如下：



```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
          package="com.perseus.a"
　　　　　 android:versionCode="1"
　　　　　 android:versionName="1.0"
          android:sharedUserId="com.share"
>
```

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
          package="com.perseus.b"
　　　　　 android:versionCode="1"
　　　　　 android:versionName="1.0"
          android:sharedUserId="com.share"
>
```

这样我们就可以在a程序中通过跳转activity的形式访问b中的数据了。

  这样的话你也许会有疑问，如果让其他的开发这知道了我们的shareUserId知道了我们的ID，那我们的数据不是暴露了，放心吧google不会犯这样的低级错误的，我们要使不同的程序能够相互访问，还需要拥有相同的签名，每个公司或者开发者的签名是唯一的，这样我们就不用担心了，另外两者能够访问，别忘了权限