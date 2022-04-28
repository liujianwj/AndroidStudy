https://blog.csdn.net/weixin_47933729/article/details/112349766

- Android中进程和线程的关系,区别

  > 进程一般是通过fork创建，线程是通过new thread
  >
  > 线程是CPU调度的最小单元，同时线程是一种有限的系统资源；而进程一般指一个执行单元，在PC和移动设备上指一个程序或者一个应用。
  > 进程有自己独立的地址空间，而进程中的线程共享此地址空间，都可以并发执行。
  > 一般来说，一个App程序至少有一个进程，一个进程至少有一个线程（包含与被包含的关系），通俗来讲就是，在App这个工厂里面有一个进程，线程就是里面的生产线，但主线程（即主生产线）只有一条，而子线程（即副生产线）可以有多个。

- 为何需要进行IPC,多进程通信可能会出现什么问题

  > 因为所有运行在不同进程的四大组件（Activity、Service、Receiver、ContentProvider）共享数据都会失败，这是由于Android为每个应用分配了独立的虚拟机，不同的虚拟机在内存分配上有不同的地址空间，这会导致在不同的虚拟机中访问同一个类的对象会产生多份副本。比如常用例子（通过开启多进程获取更大内存空间、两个或者多个应用之间共享数据、微信全家桶）。为了解决数据共享问题需要进行IPC。
  >
  > 一般来说，使用多进程通信会造成如下几方面的问题:
  >
  > - 静态成员和单例模式完全失效：独立的虚拟机造成。
  > - 线程同步机制完全失效：独立的虚拟机造成。
  > - SharedPreferences的可靠性下降：这是因为Sp不支持两个进程并发进行读写，有一定几率导致数据丢失。
  > - Application会多次创建：Android系统在创建新的进程时会分配独立的虚拟机，所以这个过程其实就是启动一个应用的过程，自然也会创建新的Application。

- Android中IPC方式有几种、各种方式优缺点

  > <img src="/Users/liujian/Documents/study/books/AndroidStudy/图片/image-20220221142238690.png" alt="image-20220221142238690" style="zoom:50%;" />
  
- 为何新增Binder来作为主要的IPC方式

  > 在讨论这个问题之前，我们知道Android也是基于Linux内核，Linux现有的进程通信手段有以下几种：
  >
  > - 管道：在创建时分配一个page大小的内存，缓存区大小比较有限；
  > - 消息队列：信息复制两次，额外的CPU消耗；不合适频繁或信息量大的通信；
  > - 共享内存：无须复制，共享缓冲区直接附加到进程虚拟地址空间，速度快；但进程间的同步问题操作系统无法实现，必须各进程利用同步工具解决；
  > - 套接字：作为更通用的接口，传输效率低，主要用于不同机器或跨网络的通信；
  > - 信号量：常作为一种锁机制，防止某进程正在访问共享资源时，其他进程也访问该资源。因此，主要作为进程间以及同一进程内不同线程之间的同步手段。 不适用于信息交换，更适用于进程中断控制，比如非法内存访问，杀死某个进程等；
  >
  > 既然有现有的IPC方式，为什么重新设计一套Binder机制呢。主要是出于三个方面的考量：
  >
  > - 效率：传输效率主要影响因素是内存拷贝的次数，拷贝次数越少，传输速率越高。从Android进程架构角度分析：对于消息队列、Socket和管道来说，数据先从发送方的缓存区拷贝到内核开辟的缓存区中，再从内核缓存区拷贝到接收方的缓存区，一共两次拷贝。
  >   而对于Binder来说，数据从发送方的缓存区拷贝到内核的缓存区，而接收方的缓存区与内核的缓存区是映射到同一块物理地址的，节省了一次数据拷贝的过程，如图：
  > - 稳定性：共享内存不需要拷贝，Binder的性能仅次于共享内存。共享内存的性能优于Binder，那为什么不采用共享内存呢，因为共享内存需要处理并发同步问题，容易出现死锁和资源竞争，稳定性较差。Socket虽然是基于C/S架构的，但是它主要是用于网络间的通信且传输效率较低。Binder基于C/S架构 ，Server端与Client端相对独立，稳定性较好。
  > - 安全性：传统Linux IPC的接收方无法获得对方进程可靠的UID/PID，从而无法鉴别对方身份；而Binder机制为每个进程分配了UID/PID，且在Binder通信时会根据UID/PID进行有效性检测。

- 什么是Binder

  > 1、binder 是一种进程间通信的机制
  >
  > 2、binder是一个虚拟物理内存驱动
  >
  > 3、binder是一个能发起通信的java类

- Binder的原理

  > Linux系统将一个进程分为用户空间和内核空间。对于进程之间来说，用户空间的数据不可共享，内核空间的数据可共享，为了保证安全性和独立性，一个进程不能直接操作或者访问另一个进程，即Android的进程是相互独立、隔离的，这就需要跨进程之间的数据通信方式。普通的跨进程通信方式一般需要2次内存拷贝，如下图所示
  >
  > <img src="/Users/liujian/Documents/study/books/AndroidStudy/图片/image-20220221144026394.png" alt="image-20220221144026394" style="zoom:30%;" />
  >
  > 一次完整的 Binder IPC 通信过程通常是这样：
  >
  > - 首先 Binder 驱动在内核空间创建一个数据接收缓存区。
  >
  > - 接着在内核空间开辟一块内核缓存区，建立内核缓存区和内核中数据接收缓存区之间的映射关系，以及内核中数据接收缓存区和接收进程用户空间地址的映射关系。(Binder IPC 机制中涉及到的内存映射通过 mmap() 来实现，mmap() 是操作系统中一种内存映射的方法。内存映射简单的讲就是将用户空间的一块内存区域映射到内核空间。映射关系建立后，用户对这块内存区域的修改可以直接反应到内核空间；反之内核空间对这段区域的修改也能直接反应到用户空间。)
  >
  > - 发送方进程通过系统调用 copyfromuser() 将数据 copy 到内核中的内核缓存区，由于内核缓存区和接收进程的用户空间存在内存映射，因此也就相当于把数据发送到了接收进程的用户空间，这样便完成了一次进程间的通信。
  >
  >   <img src="/Users/liujian/Library/Application Support/typora-user-images/image-20220221144245527.png" alt="image-20220221144245527" style="zoom:30%;" />
  >
  >   数据接收缓冲区映射如何实现？
  >
  >   Binder驱动实现了mmap()系统调用，这对字符设备是比较特殊的，因为mmap()通常用在有物理存储介质的文件系统上，而象Binder这样没有物理介质，纯粹用来通信的字符设备没必要支持mmap()。Binder驱动当然不是为了在物理介质和用户空间做映射，而是用来创建数据接收的缓存空间。先看mmap()是如何使用的：
  >
  >   ```java
  >   fd = open("/dev/binder", O_RDWR);
  >   
  >   mmap(NULL, MAP_SIZE, PROT_READ, MAP_PRIVATE, fd, 0);
  >   ```
  >
  >   这样Binder的接收方就有了一片大小为MAP_SIZE的接收缓存区。mmap()的返回值是内存映射在用户空间的地址，不过这段空间是由驱动管理，用户不必也不能直接访问（映射类型为PROT_READ，只读映射）。
  >   mmap()分配的内存除了映射进了接收方进程里，还映射进了内核空间。所以调用copy_from_user()将数据拷贝进内核空间也相当于拷贝进了接收方的用户空间，这就是Binder只需一次拷贝的‘秘密’。

- Binder Driver 如何在内核空间中做到一次拷贝的？

  > 在服务端中，通过mmap，在内核物理空间开辟一块内存，内核空间的虚拟地址，和服务端的虚拟地址都指向这一块内存，
  >
  > copy_from_user的时候，其实是往这这一块物理内存中copy数据，服务端读取的时候，则不需要copy_ to_uesr，可以直接访问。服务端给客户端传数据也是一样的，不需要copy_from_user，直接写进共享物理内存，客户端读取的时候，copy_ to_uesr是从这块内存中读取。

- 使用Binder进行数据传输的具体过程

- Bundle传递对象为什么需要序列化？Serialzable和Parcelable的区别？

  > 因为bundle传递数据时只支持基本数据类型，所以在传递对象时需要序列化转换成可存储或可传输的本质状态（字节流）。序列化后的对象可以在网络、IPC（比如启动另一个进程的Activity、Service和Reciver）之间进行传输，也可以存储到本地。
序列化实现的两种方式：实现Serializable/Parcelable接口。不同点如图：
<img src="/Users/liujian/Documents/study/books/AndroidStudy/图片/image-20220221142801351.png" alt="image-20220221142801351" style="zoom:50%;" />

- Binder框架中ServiceManager的作用

  >  Binder框架 是基于 C/S 架构的。由一系列的组件组成，包括 Client、Server、ServiceManager、Binder驱动，其中 Client、Server、Service Manager 运行在用户空间，Binder 驱动运行在内核空间。如下图所示：
  >
  >  <img src="/Users/liujian/Documents/study/books/AndroidStudy/图片/image-20220221150019240.png" alt="image-20220221150019240" style="zoom:30%;" />
  >
  >  - Server&Client：服务器&客户端。在Binder驱动和Service Manager提供的基础设施上，进行Client-Server之间的通信。
  >  - ServiceManager（如同DNS域名服务器）服务的管理者，将Binder名字转换为Client中对该Binder的引用，使得Client可以通过Binder名字获得Server中Binder实体的引用。
  >  - Binder驱动（如同路由器）：负责进程之间binder通信的建立，计数管理以及数据的传递交互等底层支持。
  >
  >  Service Manager是Binder进程间通信的核心之一，它扮演着Binder进程间通信机制上下文管理者的角色，同时负责管理系统的Service组件，并且向Client组件提供获取Service代理对象的服务。
  >  Service Manager运行在一个独立进程中，因此和Client和Service通信也是运用Binder进程间通信机制来交互的。ServiceManager 和其他进程同样采用 Bidner 通信，ServiceManager 是 Server 端，有自己的 Binder 实体，其他进程都是 Client，需要通过这个 Binder 的引用来实现 Binder 的注册，查询和获取。ServiceManager 提供的 Binder 比较特殊，它没有名字也不需要注册。当一个进程使用 BINDERSETCONTEXT_MGR 命令将自己注册成 ServiceManager 时 Binder 驱动会自动为它创建 Binder 实体（这就是那只预先造好的那只鸡）。其次这个 Binder 实体的引用在所有 Client 中都固定为 0 而无需通过其它手段获得。也就是说，一个 Server 想要向 ServiceManager 注册自己的 Binder 就必须通过这个 0 号引用和 ServiceManager 的 Binder 通信。
  >  Service Manager主要是负责如下：
  >
  >  1. 打开和映射Binder设备文件；
  >  2. 注册为Binder上下文管理者；
  >  3. 循环等待Client进程请求；
  >  4. 管理ServiceManager代理对象；
  >  5. Service组件创建，同时需要在注册ServiceManager；
  >  6. 管理Service代理对象；
  >
  >  总结：
  >
  >  ![image-20220221150415667](/Users/liujian/Documents/study/books/AndroidStudy/图片/image-20220221150415667.png)
  >
  >  Binder进程间通信模型
  >
  >  <img src="/Users/liujian/Documents/study/books/AndroidStudy/图片/image-20220221150512761.png" alt="image-20220221150512761" style="zoom:50%;" />
  >
  >  Client进程组件、Service进程组件和ServiceManager运行在用户空间，而Binder驱动程序运行在Linux系统内核空间，其中Client进程组件、Service进程组件和Service Manager均是通过调用Open、mmap和ioctl来访问虚拟设备文件/dev/binder，从而实现与Binder驱动程序的交互，进而能够间接地执行进程通信。
  >  Binder就是一种把这四个组件粘合在一起的粘结剂了，其中，核心组件便是Binder驱动程序了，Service Manager提供了辅助管理的功能，Client和Server正是在Binder驱动和Service Manager提供的基础设施上，进行Client-Server之间的通信。

- 什么是AIDL

  > AIDL(Android Interface Definition Language，Android接口定义语言)：如果在一个进程中要调用另一个进程中对象的方法，可使用AIDL生成可序列化的参数，AIDL会生成一个服务端对象的代理类，通过它客户端可以实现间接调用服务端对象的方法。
  > AIDL的本质是系统提供了一套可快速实现Binder的工具。

- AIDL使用的步骤

  > 1、新建.aidl文件，编写接口
  >
  > ```java
  > //IMyAidlInterface.aidl
  > interface IMyAidlInterface {
  > 
  >  String getName();
  > 
  > }
  > ```
  >
  > 2、build工程，自动生成文件
  >
  > ![image-20220221111154746](/Users/liujian/Documents/study/books/AndroidStudy/图片/image-20220221111154746.png)
  >
  > 3、实现service类，在onbind  方法中返回实现Stub的binder类，记得在manifest中注册
  >
  > ```java
  > class MyService : Service() {
  >  override fun onCreate() {
  >      super.onCreate()
  >  }
  > 
  >  override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
  >      return super.onStartCommand(intent, flags, startId)
  >  }
  > 
  >  override fun onDestroy() {
  >      super.onDestroy()
  >  }
  > 
  >  override fun onBind(intent: Intent?): IBinder {
  >      return MyBinder()
  >  }
  > 
  >  override fun unbindService(conn: ServiceConnection) {
  >      super.unbindService(conn)
  >  }
  > 
  >  class MyBinder: IMyAidlInterface.Stub(){
  > 
  >      override fun getName(): String {
  >          return "liujian"
  >      }
  >  }
  > }
  > 
  > 
  > //AndroidManifest.xml
  >      <service
  >          android:name=".MyService"
  >          android:enabled="true"
  >          android:exported="true">
  >          <intent-filter>
  >              <action android:name="com.jian.service.MyService" />
  >              <category android:name="android.intent.category.DEFAULT" />
  >          </intent-filter>
  >      </service>
  > ```
  >
  > 4、启动服务暴露给客户端使用
  >
  > ```kotlin
  > val intent = Intent()
  > intent.action = "com.jian.service.MyService"
  > intent.setPackage("com.jian.ktdemo")
  > startService(intent)
  > ```
  >
  > 5、将.aidl文件copy到客户端，报名路径都要一致，同样编译生成对应的aidljava文件
  >
  > 6、通过bindService找到对应的服务，建立连接，在ServiceConnection的onServiceConnected回调中获取代理对象，通过该对象调用方法访问服务端
  >
  > ```kotlin
  > class ClientActivity: AppCompatActivity() {
  > 
  >  var serviceAidlInterface: IMyAidlInterface? = null
  > 
  >  private val connection = object :ServiceConnection {
  > 
  >      override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
  >          serviceAidlInterface = IMyAidlInterface.Stub.asInterface(service)
  >      }
  > 
  >      override fun onServiceDisconnected(name: ComponentName?) {
  >      }
  >  }
  > 
  >  override fun onCreate(savedInstanceState: Bundle?) {
  >      super.onCreate(savedInstanceState)
  >      setContentView(R.layout.activity_client)
  >  }
  > 
  > fun get(view: View) {
  >      val name = serviceAidlInterface?.name
  >      Toast.makeText(this, "name--$name", Toast.LENGTH_SHORT).show()
  >  }
  > 
  >  fun connect(view: View){
  >      val intent = Intent()
  >      intent.action = "com.jian.service.MyService"
  >      intent.setPackage("com.jian.ktdemo")
  >      bindService(intent, connection, BIND_AUTO_CREATE)
  >  }
  > }
  > ```
  >
  > 

- AIDL支持哪些数据类型

  > AIDL支持的数据类型分为如下几种：
  >
  > - 八种基本数据类型：byte、char、short、int、long、float、double、boolean
  > - String，CharSequence
  > - 实现了Parcelable接口的数据类型
  > - List 类型。List承载的数据必须是AIDL支持的类型，或者是其它声明的AIDL对象
  > - Map类型。Map承载的数据必须是AIDL支持的类型，或者是其它声明的AIDL对象
  >
  > 如果要支持Map类型，则只能以Map类型出现而不能以Map的泛型类型出现（例如Map<int, [boolean](https://so.csdn.net/so/search?q=boolean&spm=1001.2101.3001.7020)>等）
  >
  > 另外，如果Map类型要出现在返回值中，需要申明out修饰

- AIDL的关键类，方法和工作流程

  > AIDL接口：继承IInterface。
  > Stub类：Binder的实现类，服务端通过这个类来提供服务。
  > Proxy类：服务端的本地代理，客户端通过这个类调用服务端的方法。
  > asInterface()：客户端调用，将服务端返回的Binder对象，转换成客户端所需要的AIDL接口类型的对象。如果客户端和服务端位于同一进程，则直接返回Stub对象本身，否则返回系统封装后的Stub.proxy对象。
  > asBinder()：根据当前调用情况返回代理Proxy的Binder对象。
  > onTransact()：运行在服务端的Binder线程池中，当客户端发起跨进程请求时，远程请求会通过系统底层封装后交由此方法来处理。
  > transact()：运行在客户端，当客户端发起远程请求的同时将当前线程挂起。之后调用服务端的onTransact()直到远程请求返回，当前线程才继续执行。
  >
  > 工作流程：客户端通过bindService请求服务，在serviceConnection中的onServiceConnected回调中，通过调用IMyAidlInterface.Stub.asInterface(service)，返回代理类Proxy。
  >
  > 在调用的时候，走Proxy代理类对应的方法，方法中会对数据进行包装下，并创建一个reply包接收返回数据，然后通过binder机制，进入到服务端的Stud.onTransact 方法中，通过code查找到对应的方法，然后调用。如果有返回值通过reply写回。

- 如何优化多模块都使用AIDL的情况

  > - 当有多个业务模块都需要AIDL来进行IPC，此时需要为每个模块创建特定的aidl文件，那么相应的Service就会很多。必然会出现系统资源耗费严重、应用过度重量级的问题。解决办法是建立Binder连接池，即将每个业务模块的Binder请求统一转发到一个远程Service中去执行，从而避免重复创建Service。
  > - 原理：每个业务模块创建自己的AIDL接口并实现此接口，然后向服务端提供自己的唯一标识和其对应的Binder对象。服务端只需要一个Service并提供一个queryBinder接口，它会根据业务模块的特征来返回相应的Binder对象，不同的业务模块拿到所需的Binder对象后就可以进行远程方法的调用了。
  >
  > ```java
  >      private final BookController.Stub stub = new BookController.Stub() {
  > 
  >         @Override
  >         public List<Book> getBookList() throws RemoteException {
  >             return bookList;
  >         }
  > 
  >         @Override
  >         public void addBookInOut(Book book) throws RemoteException {
  >             if (book != null) {
  >                 book.setName("服务器改了新书的名字 InOut");
  >                 bookList.add(book);
  >             } else {
  >                 Log.e(TAG, "接收到了一个空对象 InOut");
  >             }
  >         }
  > 
  >     };
  >     ...
  > 
  >     @Override
  >     public IBinder onBind(Intent intent) {
  >         //客户端bindService时传递
  >         String type = intent.getStringExtra("type")
  >         if("one".isEqual(type)){
  >             return stub;
  >         }else{
  >             return stub1;
  >         }
  >         ...
  >     }
  > ```

- 使用 Binder 传输数据的最大限制是多少，被占满后会导致什么问题

  > 同步中：一M减8k，异步是/2，，，异步很少使用，正常的都是同步
  >
  > 原因：在底层binder驱动中通过mmap开辟一块空间映射，这个空间的大小设置的就是1 x 1024 x 1024 - 8
  >
  > 抛出异常，实际还达不到一M减8k，因为内部还有包装，包装壳也有大小

- Binder 驱动加载过程中有哪些重要的步骤

- 系统服务与bindService启动的服务的区别

  > 系统服务有对应的名字id
  >
  > bindService启动的服务是匿名的

- Activity的bindService流程

- 不通过AIDL，手动编码来实现Binder的通信