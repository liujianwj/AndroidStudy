- 什么是ContentProvider及其使用

  > ContentProvider是Android中跨进程数据交换的重要类，Android为数据交换提供了一个标准ContentProvider.
  >
  > 
  >
  > 那么应该如何完整实现开发一个contentProvider呢
  > 1、首先需要定义自己的ContentProvider，该类需要继承Android提供的ContentProvider的基类
  >
  >   ```java
  > public class DictProvider extends ContentProvider {
  >  String TAG  = "=======TAG=====" ;
  >  @Override
  >  public boolean onCreate() {
  >      Log.v(TAG,"onCreate");
  >      return false;
  >  }
  >  @Override
  >  public Cursor query(@NonNull Uri uri, @Nullable String[] projection, @Nullable String selection, @Nullable String[] selectionArgs, @Nullable String sortOrder) {
  >      Log.v(TAG,"query");
  >      return null;
  >  }
  >  @Override
  >  public String getType(@NonNull Uri uri) {
  >      Log.v(TAG,"getType");
  >      return null;
  >  }
  >  @Override
  >  public Uri insert(@NonNull Uri uri, @Nullable ContentValues values) {
  >      Log.v(TAG,"insert");
  >      return null;
  >  }
  > 
  >  @Override
  >  public int delete(@NonNull Uri uri, @Nullable String selection, @Nullable String[] selectionArgs) {
  >      Log.v(TAG,"delete");
  >      return 0;
  >  }
  > 
  >  @Override
  >  public int update(@NonNull Uri uri, @Nullable ContentValues values, @Nullable String selection, @Nullable String[] selectionArgs) {
  >      Log.v(TAG,"update");
  >      return 0;
  >  }
  > }
  > 
  >   ```
  >
  > 
  >
  > 2、需要在AndroidManifest.xml文件中注册这个ContentProvider
  >
  > ```java
  > <provider
  >          android:authorities="com.jewelermobile.gangfu.zdydemo1.DictProvider"
  >          android:name=".activity.DictProvider"
  >          android:exported="true"/>// true 允许外部应用启动 反之 不能启动
  > ```

- ContentProvider的权限管理

  > 1. 无限制：
  >
  >    如果一个provider 用了exported=true，那么任意三方调用方都能对其进行调用；
  >
  > 2. 只对满足特定条件三方才进行暴露：
  >
  >    provider方：
  >
  >    <provider
  >        android:name=".MyContentProvider"
  >        android:authorities="com.anddle.mycontentprovider"
  >        android:enabled="true"
  >        android:exported="true"
  >        android:permission="com.anddle.provideraccess" />
  >
  >       <permission
  >            android:name="com.anddle.provideraccess"
  >            android:label="provider pomission"
  >            android:protectionLevel="normal" />
  >
  >    1).配置清单中加android:permission="com.anddle.provideraccess" 
  >
  >    2).再定义permission，其对应的name与声明的权限对应
  >
  >    调用方：
  >
  >    清单文件中声明： <uses-permission android:name="com.anddle.provideraccess" /> 即可调用provider
  >
  > 3. 在②只对满足特定条件基础上并区分读写
  >
  >    provider方：
  >
  >    <provider
  >        android:name=".MyContentProvider"
  >        ......
  >        android:readPermission="com.anddle.provideraccess.read" />
  >
  >    <permission
  >            android:name="com.anddle.provideraccess.read"
  >            android:label="provider pomission"
  >            android:protectionLevel="normal" />
  >
  >    1.配置清单中加android:readpermission(writepermission)="com.anddle.provideraccess.read" 
  >
  >    2.再定义permission，其对应的name与声明的权限对应
  >
  >    调用方：
  >
  >    清单文件中声明： <uses-permission android:name="com.anddle.provideraccess.read" /> 即可调用provider

- ContentProvider,ContentResolver,ContentObserver之间的关系

  >ContentProvider: 四大组件之一，内容提供者，使用`ContentProvider`对外共享数据的好处是统一了数据的访问方式。
  >
  >- 四大组件的内容提供者，主要用于对外提供数据
  >- 实现各个应用程序之间的（跨应用）数据共享，比如联系人应用中就使用了ContentProvider（我在上篇ContentProvider是如何实现数据共享的？—— ContentProvider数据共享案例中提供了一个简单的删除联系人的案例）。其实它也只是一个中间人，真正的数据源是文件或者SQLite等。
  >- 一个应用实现`ContentProvider`来提供内容给别的应用来操作，通过`ContentResolver`来操作别的应用数据，当然在自己的应用中也可以。
  >- 负责管理结构化数据的访问；
  >- 封装数据并且提供一套定义数据安全的机制；
  >- 是一套在不同进程间进行数据访问的接口；
  >- 为数据跨进程访问提供了一套安全的访问机制，对数据组织和安全访问提供了可靠的保证；
  >
  >
  >
  >ContentResolver: ContentResolver可以不同URI操作不同的ContentProvider中的数据，外部进程可以通过ContentResolver与ContentProvider进行交互。
  >
  >- 内容解析者，用于获取内容提供者提供的数据
  >
  >- ContentResolver.notifyChange（uri）发出消息
  >
  >
  >
  >ContentObserver: 观察ContentProvider中的数据变化，并将变化通知给外界。
  >
  >- 内容监听器，可以监听数据的改变状态
  >
  >- 目的是观察（捕捉）特定Uri引起的数据库的变化，继而做一些相应的处理，它类似于数据库技术中的触发器（Trigger），当ContentObserver所观察的Uri发生变化时，便会触发它。触发器分为表触发器、行触发器，相应地ContentObsever也分为表ContentObserver、行ContentObserver，当然这是与它所监听的Uri MIME Type有关的
  >
  >- ContentResolver.registerContentObserver()监听消息

  

- ContentProvider的实现原理

  > 1. application初始化的时候会installProvider
  > 2. 向AMS请求provider的时候如果对端进程不存在则请求的那个线程需要一直等待
  > 3. 当对方的进程启动之后并publish之后，请求provider的线程才可返回，所以尽量不要在主线程请求provider
  > 4. 请求provider分为stable以及unsbale，stable类型链接在server进程挂掉之后，client进程会跟着被株连kill
  > 5. insert/delete/update默认建立stable链接，query默认建立unstable链接，如果失败会重新建立stable链接
  > 6. AMS作为一个中间管理员的身份，所有的provider会向它注册
  > 7. 向AMS请求到provider之后，就可以在client和server之间自行binder通信，不需要再经过systemserver

- ContentProvider的优点

  > 1.封装数据，提供统一接口，当项目需求需要修改数据源的时候，节省时间和人力
  >
  > 2.提供一种跨进程数据共享的方式
  >
  > 3.数据更新通知机制。

- Uri 是什么

  > Content://com.jewelermobile.gangfu.zdydemo1.DictProvider/资源部分
  >
  > （1）Content:// 这部分是Android的ContentProvider规定，并且固定
  > （2）com.jewelermobile.gangfu.zdydemo1.DictProvider 这部分是注册时的android:authorities，注册之后并且固定
  > （3）资源部分，这个根据查询资源改变所以不固定
  >
  > Uri其实就是一个地址，只有通过Uri才能对应去操作ContentProvider，就像一条唯一通往胜利道路。