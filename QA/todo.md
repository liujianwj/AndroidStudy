https://juejin.cn/post/6990332687063990286

http://www.xiangxueketang.cn/enjoy/removal/article_7?bd_vid=9470290798457506851

> 内容概要：包括 Android、Java、数据结构与算法、计算机网路四大模块、

> Android内含：Activity、Fragment、service、布局优化、AsyncTask相关、Android 事件分发机制、 Binder、Android 高级必备 ：AMS,WMS,PMS、Glide、 Android 组件化与插件化等面试题和技术栈！

> Java内含：HashMap、ArrayList、LinkedList 、Hashset 源码分析、内存模型、垃圾回收算法（JVM）、多线程、注解、反射、泛型、设计模式等面试题和技术栈！

> 

### 一. Android相关




#### 5.Handler



#### 6.View绘制



#### 8.RecycleView

- RecyclerView的多级缓存机制,每一级缓存具体作用是什么,分别在什么场景下会用到哪些缓存
- RecyclerView的滑动回收复用机制
- RecyclerView的刷新回收复用机制
- RecyclerView 为什么要预布局
- ListView 与 RecyclerView区别
- RecyclerView性能优化

#### 9.Viewpager&Fragment

- Fragment的生命周期 & 结合Activity的生命周期
- Activity和Fragment的通信方式， Fragment之间如何进行通信
- getFragmentManager、getSupportFragmentManager 、getChildFragmentManager之间的区别？
- 为什么使用Fragment.setArguments(Bundle)传递参数
- FragmentPagerAdapter与FragmentStatePagerAdapter的区别与使用场景
- FragmentPageAdapter和FragmentStatePageAdapter区别及使用场景
- Fragment懒加载
- ViewPager2与ViewPager区别
- Fragment嵌套问题

#### 10.WebView

- 如何提高WebView加载速度
- WebView与 js的交互
- WebView的漏洞
- JsBridge原理

#### 11.动画

- 动画的类型

  > 帧动画、补间动画、属性动画

- 补间动画和属性动画的区别

  > 补间动画只是视觉上的改变，无法改变控件的属性，点击效果问题
  >
  > 属性动画是改变控件的属性，但是3.0以下不支持

- ObjectAnimator，ValueAnimator及其区别

  > 

- TimeInterpolator插值器，自定义插值器

- TypeEvaluator估值器

#### 12.Bitmap

- Bitmap 内存占用的计算
- getByteCount() & getAllocationByteCount()的区别
- Bitmap的压缩方式
- LruCache & DiskLruCache原理
- 如何设计一个图片加载库
- 有一张非常大的图片,如何去加载这张大图片
- 如果把drawable-xxhdpi下的图片移动到drawable-xhdpi下，图片内存是如何变的。
- 如果在hdpi、xxhdpi下放置了图片，加载的优先级。如果是400_800，1080_1920，加载的优先级。

#### 13.mvc&mvp&mvvm

- MVC及其优缺点
- MVP及其优缺点
- MVVM及其优缺点
- MVP如何管理Presenter的生命周期，何时取消网络请求

#### 14.Binder 

#### 15.内存泄漏&内存溢出

- 什么是OOM & 什么是内存泄漏以及原因
- Thread是如何造成内存泄露的，如何解决？
- Handler导致的内存泄露的原因以及如何解决
- 如何加载Bitmap防止内存溢出
- MVP中如何处理Presenter层以防止内存泄漏的

#### 16.性能优化

- 内存优化
- 启动优化
- 布局加载和绘制优化
- 卡顿优化
- 网络优化

#### 17.Window&WindowManager

- 什么是Window
- 什么是WindowManager
- 什么是ViewRootImpl
- 什么是DecorView
- Activity，View，Window三者之间的关系
- DecorView什么时候被WindowManager添加到Window中

#### 18.WMS

- 什么是WMS
- WMS是如何管理Window的
- IWindowSession是什么，WindowSession的创建过程是怎样的
- WindowToken是什么
- WindowState是什么
- Android窗口大概分为几种？分组原理是什么
- Dialog的Context只能是Activity的Context，不能是Application的Context
- App应用程序如何与SurfaceFlinger通信的
- View 的绘制是如何把数据传递给 SurfaceFlinger 的
- 共享内存的具体实现是什么
- relayout是如何向SurfaceFlinger申请Surface
- 什么是Surface

#### 19.AMS

- ActivityManagerService是什么？什么时候初始化的？有什么作用？
- ActivityThread是什么?ApplicationThread是什么?他们的区别
- Instrumentation是什么？和ActivityThread是什么关系？
- ActivityManagerService和zygote进程通信是如何实现的
- ActivityRecord、TaskRecord、ActivityStack，ActivityStackSupervisor，ProcessRecord
- ActivityManager、ActivityManagerService、ActivityManagerNative、ActivityManagerProxy的关系
- 手写实现简化版AMS

#### 20.系统启动

- android系统启动流程
- SystemServer，ServiceManager，SystemServiceManager的关系
- 孵化应用进程这种事为什么不交给SystemServer来做，而专门设计一个Zygote
- Zygote的IPC通信机制为什么使用socket而不采用binder

#### 21.App启动&打包&安装

- 应用启动流程

  > 桌面launcher点击的时候，launcher调用startActivity->startActivityForResult->instarmtation.start->AMS->检查信息，存储信息->通知launcher调用onPause方法->launcher接到信息->handler发送消息->performPause->通知AMS->检查是否已经创建进程->通知zygote进程fork出一个新进程->新进程activitythread中调用main方法，开启handler->application.attach->将applicationThread发送到wms->wms接收并保存，后面都是通过这个跟app通信->通知app调用onCreate等方法

- apk组成和Android的打包流程

  > - `AndroidManifest.xml` 描述配置文件
  >
  > - `classes.dex` 是java源码编译后生成的java字节码文件，因方法数限制，可拆分为多个dex.优化重排后生成dex文件，生成的dex文件可以在Dalvik虚拟机执行，且速度比较快
  >
  > - `META-INF` 存放签名和证书的目录
  >
  > - `res` 主要是存放图片资源
  >
  > - `lib` 主要是存放so库，各个cpu架构
  >
  > - `assets` 主要存放不需要编译处理的文件
  >
  > 
  >
  > 显示aapt将资源文件生成R.java文化，然后aidl工具将aidl文件生成java文件，和项目中的java文件一起编译成class文件，再通过dex工具，生成dex文件，，，然后结合资源文件，以及第三方的一些资源，一起打包成apk文件，然后进行签名，签名之后进行对齐

- Android的签名机制，签名如何实现的,v2相比于v1签名机制的改变

  > https://blog.csdn.net/singwhatiwanna/article/details/103142836
  >
  > <img src="/Users/liujian/Documents/study/books/AndroidStudy/图片/image-20220218143229577.png" alt="image-20220218143229577" style="zoom: 33%;" />
  >
  > 签名机制：
  >
  > 主要是为了严重apk是否被人篡改，这里主要用到数字签名--->消息摘要+非对称加密
  >
  > 首先对内容进行消息摘要，然后使用私钥对摘要信息进行加密得到签名信息，最后将签名信息和原始内容一起打进apk
  >
  > 验证的时候先对原始内容进行消息摘要，然后用公钥对之前的签名信息进行解密，，再将解密后的信息与消息摘要对比，如果一致，则表示原始内容未被篡改。。
  >
  > 原因：篡改后无法用私钥加密，因为拿不到私钥，，公钥由机构签发。
  >
  > v1,2,v3
  >
  > v1和v2机制不一样，v3是在v2上的一个升级
  >
  > v1: 在 `META-INF` 文件夹下有三个文件：`MANIFEST.MF`、`CERT.SF`、`CERT.RSA`
  >
  > v1是用摘要算法对所有文件进行消息摘要，然后base64处理后，放入MANIFEST.MF，CERT.SF中则是存储对MANIFEST.MF的消息摘要即base64 处理的结果，CERT.RSA把之前生成的 CERT.SF 文件，用私钥计算出签名, 然后将签名以及包含公钥信息的数字证书一同写入 CERT.RSA 中保存。Android APK 中的 CERT.RSA 证书是自签名的，并不需要这个证书是第三方权威机构发布或者认证的。
  >
  > v2:验证速度更快，验证内容更完整
  >
  > 从 Android 7.0 开始，Android 支持了一套全新的 V2 签名机制，为什么要推出新的签名机制呢？通过前面的分析，可以发现 v1 签名有两个地方可以改进：
  >
  > - 签名校验速度慢
  >
  > 校验过程中需要对apk中所有文件进行摘要计算，在 APK 资源很多、性能较差的机器上签名校验会花费较长时间，导致安装速度慢。
  >
  > - 完整性保障不够
  >
  > META-INF 目录用来存放签名，自然此目录本身是不计入签名校验过程的，可以随意在这个目录中添加文件，比如一些快速批量打包方案就选择在这个目录中添加渠道文件。
  >
  > 在apk文件新加一个区块，专门放签名信息、证书链等。
  >
  > 把 APK 按照 1M 大小分割，分别计算这些分段的摘要，最后把这些分段的摘要在进行计算得到最终的摘要也就是 APK 的摘要。然后将 APK 的摘要 + 数字证书 + 其他属性生成签名数据写入到 APK Signing Block 区块

- APK的安装流程

  > apk安装的四大步骤：
  >
  > （1）拷贝apk到指定的目录：默认情况下，用户安装的apk首先会拷贝到/data/app下，用户有访问/data/app目录的权限，但系统出厂的apk文件会被放到/system分区下，包括/system/app，/system/vendor/app，以及/system/priv-app等，该分区需要root权限的用户才能访问。
  > （2）加载apk、拷贝文件、创建应用的数据目录：为了加快APP的启动速度，apk在安装的时候，会首先将APP的可执行文件（dex）拷贝到/data/dalvik-cache目录下，缓存起来。再在/data/data/目录下创建应用程序的数据目录（以应用包名命令），用来存放应用的数据库、xml文件、cache、二进制的so动态库等。
  > （3）解析apk的AndroidManifest.xml文件：在安装apk的过程中，会解析apk的AndroidManifest.xml文件，将apk的权限、应用包名、apk的安装位置、版本、userID等重要信息保存在/data/system/packages.xml文件中。这些操作都是在PackageManagerService中完成
  > 的。
  > （4）显示icon图标：应用程序经过PMS中的逻辑处理后，相当于已经注册好了，如果想要在Android桌面上看到icon图标，则需要Launcher将系统中已经安装的程序展现在桌面上。
  >
  > 
  >
  > 总结APK的安装流程如下：
  >
  > 复制APK安装包到/data/app目录下，解压缩并扫描安装包，向资源管理器注入APK资源，解析AndroidManifest文件，并在/data/data目录下创建对应的应用数据目录，然后针对Dalvik/ART环境优化dex文件，保存到dalvik-cache目录，将AndroidManifest文件解析出的组件、权限注册到PackageManagerService并发送广播。
  >
  > https://blog.csdn.net/mysimplelove/article/details/93619361

#### 22.序列化

- 什么是序列化
- 为什么需要使用序列化和反序列化
- 序列化的有哪些好处
- Serializable 和 Parcelable 的区别
- 什么是serialVersionUID
- 为什么还要显示指定serialVersionUID的值?

#### 23.模块化&组件化

- 什么是模块化
- 什么是组件化
- 组件化优点和方案
- 组件独立调试
- 组件间通信
- Aplication动态加载
- ARouter原理

#### 24.热修复&插件化

- 插件化的定义
- 插件化的优势
- 插件化框架对比
- 插件化流程
- 插件化类加载原理
- 插件化资源加载原理
- 插件化Activity加载原理
- 热修复和插件化区别
- 热修复原理

#### 25.AOP

- AOP是什么
- AOP的优点
- AOP的实现方式,APT,AspectJ,ASM,epic,hook

#### 26.Jectpack

- Navigation
- DataBinding
- Viewmodel
- livedata
- liferecycle

#### 27.开源框架

- Okhttp源码流程,线程池

- Okhttp拦截器,addInterceptor 和 addNetworkdInterceptor区别

- Okhttp责任链模式

- Okhttp缓存怎么处理

- Okhttp连接池和socket复用

- Glide怎么绑定生命周期

- Glide缓存机制,内存缓存，磁盘缓存

- Glide与Picasso的区别

- LruCache原理

- Retrofit源码流程,动态代理

- LeakCanary弱引用,源码流程

- Eventbus

- Rxjava

### 二. Java相关

#### 1.HashMap

- HashMap原理

  > HashMap的工作原理 ：HashMap是基于散列法（又称哈希法）的原理，使用put(key, value)存储对象到HashMap中，使用get(key)从HashMap中获取对象。当我们给put()方法传递键和值时，我们先对键调用hashCode()方法，返回的hashCode用于找到bucket（桶）位置来储存Entry对象。HashMap是在bucket中储存键对象和值对象，作为Map.Entry。并不是仅仅只在bucket中存储值。

- HashMap 有用过吗？您能给我说说他的主要用途吗？

  > 有用过，我在平常工作中经常会用到HashMap 这种数据结构，HashMap 是基于Map 接口实现的一种键-值对<key,value>的存储结构，允许null 值，同时非有序，非同步(即线程不安全)。HashMap 的底层实现是数组+ 链表+ 红黑树（JDK1.8 增加了红黑树部分）。它存储和查找数据时，是根据键key 的hashCode的值计算出具体的存储位置。HashMap 最多只允许一条记录的键key 为null，HashMap 增删改查等常规操作都有不错的执行效率，是ArrayList 和LinkedList等数据结构的一种折中实现。

- 您能说说 HashMap 常用操作的底层实现原理吗？如存储 put(K key, V value)，查找 get(Object key)，删除 remove(Object key)，修改 replace(K key, V value)等操作

- hash 冲突（或者叫 hash 碰撞）是什么？为什么会出现这种现象，如何解 决 hash 冲突？

- HashMap 的容量为什么一定要是 2 的 n 次方？

- 您能说说 HashMap 和 HashTable 的区别吗？

  > Hashtable 是早期Java类库提供的一个哈希表实现，本身是同步的，不支持 null 键和值，由于同步导致的性能开销，所以已经很少被推荐使用。
  >
  > HashMap与 HashTable主要区别在于 HashMap 不是同步的，支持 null 键和值等。通常情况下，HashMap 进行 put 或者 get 操作，可以达到常数时间的性能，所以它是绝大部分利用键值对存取场景的首选。
  >
  > TreeMap 则是基于红黑树的一种提供顺序访问的 Map，和 HashMap 不同，它的 get、put、remove 之类操作都是 O（log(n)）的时间复杂度，具体顺序可以由指定的 Comparator 来决定，或者根据键的自然顺序来判断。

- HashMap中put()如何实现的

  > ①.判断键值对数组table[i]是否为空或为null，否则执行resize()进行扩容；
  >
  > ②.根据键值key计算hash值得到插入的数组索引i，如果table[i]==null，直接新建节点添加，转向⑥，如果table[i]不为空，转向③；
  >
  > ③.判断table[i]的首个元素是否和key一样，如果相同直接覆盖value，否则转向④，这里的相同指的是hashCode以及equals；
  >
  > ④.判断table[i] 是否为treeNode，即table[i] 是否是红黑树，如果是红黑树，则直接在树中插入键值对，否则转向⑤；
  >
  > ⑤.遍历table[i]，判断链表长度是否大于8，大于8的话把链表转换为红黑树，在红黑树中执行插入操作，否则进行链表的插入操作；遍历过程中若发现key已经存在直接覆盖value即可；
  >
  > ⑥.插入成功后，判断实际存在的键值对数量size是否超多了最大容量threshold，如果超过，进行扩容。

- HashMap中get()如何实现的

  > ①.指定key 通过hash函数得到key的hash值
  > int hash=key.hashCode();
  >
  > ②.调用内部方法 getNode()，得到桶号(一般为hash值对桶数求模)
  > int index =hash%Entry[].length;
  > jdk1.6版本后使用位运算替代模运算，int index=hash&( Entry[].length - 1）;
  >
  > ③.比较桶的内部元素是否与key相等，若都不相等，则没有找到。相等，则取出相等记录的value。
  >
  > ④.如果得到 key 所在的桶的头结点恰好是红黑树节点，就调用红黑树节点的 getTreeNode() 方法，否则就遍历链表节点。getTreeNode 方法使通过调用树形节点的 find()方法进行查找。由于之前添加时已经保证这个树是有序的，因此查找时基本就是折半查找，效率很高。
  >
  > ⑤.如果对比节点的哈希值和要查找的哈希值相等，就会判断 key 是否相等，相等就直接返回；不相等就从子树中递归查找。

- 为什么HashMap线程不安全

  >  没有加锁

- HashMap1.7和1.8有哪些区别

  > 1.7数据结构是数组+链表，当发生hash冲突时会以链表的实现添加到尾部，如果冲突频繁发生，需要非常都遍历链表，，时间复杂度变成O(n)，所以到了1.8是优化成，当整个数量满64，且单个链表数量满8时，将数据结构转化成

- 解决hash冲突的时候，为什么用红黑树

  > 查找效率高

- 红黑树的效率高，为什么一开始不用红黑树存储

  > 因为红黑树需要进行左旋，右旋操作， 而单链表不需要，
  > 以下都是单链表与红黑树结构对比。
  > 如果元素小于8个，查询成本高，新增成本低
  > 如果元素大于8个，查询成本低，新增成本高

- 不用红黑树，用二叉查找树可以不

  > 红黑树适用于大量插入和删除；因为它是非严格的平衡树；只要从根节点到叶子节点的最长路径不超过最短路径的2倍，就不进行平衡调节
  > AVL树是严格的平衡树，上述的最短路径与最长路径的差不能超过|1|，AVL允许的差值小；在进行大量插入和删除操作时，频繁地进行平衡调整会严重降低效率；
  > 红黑树虽然不是严格的平衡树，但是其依旧是平衡树；查找效率是logn；
  > AVL也是logn；
  > 红黑树舍去了严格的平衡，使其插入，删除，查找的效率稳定在O(logn)
  > 反观AVL树，查找没问题O(logn)，但是为了保证高度平衡，动态插入和删除的代价也随之增加，综合效率肯定达不到O(logn)
  > 所以在进行大量插入，删除操作时，红黑树更优一些

- 为什么阀值是8才转为红黑树

  > 注释中有以下几点我们需要关注：
  >
  > - TreeNodes 对象所占内存是 普通Nodes 对象的 2倍；
  > - 理想情况下，随机哈希码和负载因子为0.75的情况下，桶中个数出现的频率服从泊松分布；
  >
  > > 泊松分布的公式：P(X=k) = exp(-λ) * pow(λ, k) / k!
  > >  当 λ = 0.5 时，算出的概率如上！链表长度达到8的概率为0.00000006，再之后则为千万分之一！
  >
  > 因此，当链表元素个数为8时，就将链表转化为红黑树。

- 为什么退化为链表的阈值是6

  > - 如果不设退化阀值，只以8来树化与退化：
  >    那么8将成为一个临界值，时而树化，时而退化，此时会非常影响性能，因此，我们需要一个比8小的退化阀值；
  > - UNTREEIFY_THRESHOLD = 7
  >    同样，与上面的情况没有好多少，仅相差1个元素，仍旧会在链表与树之间反复转化；
  > - 那为什么是6呢？
  >    源码中也说了，考虑到内存（树节点比普通节点内存大2倍，以及避免反复转化），所以，退化阀值最多为6。
  >
  > 同时，我们还需要注意一点：
  >
  > 
  >
  > ```dart
  > /**
  >  * The smallest table capacity for which bins may be treeified.
  >  * (Otherwise the table is resized if too many nodes in a bin.)
  >  * Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts
  >  * between resizing and treeification thresholds.
  >  */
  > static final int MIN_TREEIFY_CAPACITY = 64;
  > ```
  >
  > 上述注释说明了，开启树化至少是HashMap元素已经是64个时，才考虑将链表转为红黑树。
  >
  > 而红黑树初始默认大小是16，因此，在扩容至64之前，都还是采用连续数组+链表的方式来存储。

- hash冲突有哪些解决办法

  > - 扩容
  > - 加载因子 
  > - 阈值
  > - 扰动函数 https://www.zhihu.com/question/20733617

- HashMap在什么条件下扩容

  > Put的时候数量  > max * 加载因子，或者在hash冲突时，链表长度大于8，且数组长度 < 64

- HashMap中hash函数怎么实现的，还有哪些hash函数的实现方式

- 为什么不直接将hashcode作为哈希值去做取模,而是要先高16位异或低16位

  >  直接取模，那有效值永远都是低位数

- 为什么扩容是2的次幂

  > 1、当2n = length , hashCode % length = hashCode & (length - 1 )，对长度取模正好等于与长度-1， 用位移替代取模运算，提升性能。
  >
  > 2、当length为2的次幂时，length - 1对应的位数上都是1，则有效位数越多，hashCode & (length - 1 )相与后的结果可能性越多，碰撞的概率越低。

- 链表的查找的时间复杂度是多少

  > n

- 红黑树

  > 红黑树的特性:
  > （1）每个节点或者是黑色，或者是红色。
  > （2）根节点是黑色。
  > （3）每个叶子节点（NIL）是黑色。 [注意：这里叶子节点，是指为空(NIL或NULL)的叶子节点！]
  > （4）如果一个节点是红色的，则它的子节点必须是黑色的。
  > （5）从一个节点到该节点的子孙节点的所有路径上包含相同数目的黑节点。

#### 2.ArrayList

- ArrayList定义

- ArrayList 的构造器

- add 方法源码分析

- get 方法源码分析

- set 方法源码分析

- ArrayList和LinkedList的区别，以及应用场景

#### 3. LinkedList

- LinkedList 定义

- LinkedList 支持的操作

- Node 类

- addFirst 源码分析

- getFirst 方法源码分析

- removeFirst 方法源码分析

- add(int index, E e)方法源码分析
#### 4. Hashset 源码

- 属性

- 构造方法

- 添加元素

- 删除元素

- 查询元素

- 遍历元素

- 全部源码

#### 5. 内存模型

- 内存模型产生背景
- 物理机的并发问题
- Java 内存模型的组成分析
- Java 内存间的交互操作
- Java 内存模型运行规则

#### 6. 垃圾回收算法（JVM）

- Jvm的内存模型,每个里面都保存的什么
- 类加载机制的几个阶段加载、验证、准备、解析、初始化、使用、卸载
- 对象实例化时的顺序
- 类加载器,双亲委派及其优势
- 垃圾回收机制
- 谈谈对 JVM 的理解?
- JVM 内存区域，开线程影响哪块区域内存？
- 对 Dalvik、ART 虚拟机有什么了解？对比

#### 7.多线程

- 谈一谈java线程模型

  > （1）线程分为用户线程和内核线程；
  >
  > （2）线程模型有多对一模型、一对一模型、多对多模型；
  >
  > （3）操作系统一般只实现到一对一模型；
  >
  > （4）Java使用的是一对一线程模型，所以它的一个线程对应于一个内核线程，调度完全交给操作系统来处理；
  >
  > （5）Go语言使用的是多对多线程模型，这也是其高并发的原因，它的线程模型与Java中的ForkJoinPool非常类似；
  >
  > （6）python的gevent使用的是多对一线程模型；
  >
  > https://www.cnblogs.com/tong-yuan/p/11626491.html

- Java中创建线程的方式,Callable,Runnable,Future,FutureTask

- 线程的几种状态

  > Java线程共有5中状态，分别为：新建(new)、就绪(runnable)、运行(running)、堵塞(blocked)、死亡(dead)

- 谈谈线程死锁，如何有效的避免线程死锁？

- 如何实现多线程中的同步

- synchronized和Lock的使用、区别,原理；

- volatile，synchronized和volatile的区别？为何不用volatile替代synchronized？

- 锁的分类，锁的几种状态，CAS原理

- 为什么会有线程安全？如何保证线程安全

- sleep()与wait()区别,run和start的区别,notify和notifyall区别,锁池,等待池

- Java多线程通信

- 为什么Java用线程池

- Java中的线程池参数,共有几种

- 说下 Java 中的线程创建方式，线程池的工作原理

#### 8.注解

- 注解的分类和底层实现原理
- 自定义注解

#### 9.反射

- 什么是反射
- 反射机制的相关类
- 反射中如何获取Class类的实例
- 如何获取一个类的属性对象 & 构造器对象 & 方法对象
- Class.getField和getDeclaredField的区别，getDeclaredMethod和getMethod的区别
- 反射机制的优缺点

#### 10.泛型

- 泛型概念的提出（为什么需要泛型）？
- 什么是泛型？
- 自定义泛型接口、泛型类和泛型方法
- 类型通配符

#### 11.设计模式

- 你所知道的设计模式有哪些
- 单例设计模式
- 工厂设计模式
- 建造者模式（Builder）
- 适配器设计模式
- 装饰模式（Decorator）
- 策略模式（strategy）
- 观察者模式（Observer）







1. HashMap 1.7，1.8的差异，1.8中什么情况下转换为红黑树，构造函数中参数代表的意思

   > 在HashMap1.8之前，都是数组+链表的结构，当发生hash冲突时，添加到链表后面。而在1.8是在总数量大于64，且链表个数大于8时，会转成红黑树存储，方便查找。

1. 用什么Map可以保证线程安全，为什么？ConcurrentHashMap为什么能保证线程安全？1.7和1.8原理有什么差异。

   > ConcurrentHashMap是线程安全的，添加的时候，通过片段锁，就是讲数据方程多个fragment，对单个fragment进行加锁，读的话是通过volatile字段保障每次读的都是最新的。1.7/1.8，红黑树，，会使用到CAS保障线程安全。

1. 有多少种单例模式，枚举算不算单例，单例模式中不用volatile会导致什么问题？volatile特性是什么？为什么android中不推荐使用枚举。

   > 饿汉式（直接创建静态实例，利用静态变量保障线程安全）、懒汉式（当需要时，创建，线程不安全）、双层判断锁机制（先判断是否为空，然后锁住，然后再判断是否为空，再创建，同时用volatile修饰，避免内部指令优化，就是虚拟机会在保障整体逻辑正确的情况下，调整指令执行顺序，这样可能会出现判断的时候对象已经创建，但其实内部有些指令还没执行完，直接使用的时候会出错），枚举其实在内部会为每个枚举值创建一个对象，浪费内存空间。

1. Glide中怎么实现图片的加载进度条，Glide的缓存是怎么设计的？为什么要用弱引用。

   > 可以自己添加
   >
   > 磁盘缓存和内存缓存，，内存缓存分活动缓存/Lru内存缓存，活动缓存没有大小限制，使用时，先在Lru内存缓存/磁盘缓存中查找，如果找到了则移除到活动缓存，如果没找到则下载，然后添加到活动缓存，使用完之后添加到Lru内存缓存，Lru内存缓存
   >
   > 弱引用是在发生GC的时候可以回收资源，避免内存异常。

1. implementation 和 api的区别是什么？

   > implementation引用到只在当前模块下使用，上层没办法使用，api是可以依赖传递

1. 事件分发的流程，以及怎么解决滑动冲突？

   > ViewRootImpl接受到事件->DectorView.dispatcherEvent->Activity.dispatcherEvent->PhoneWindow->DectorView(super.dispatcherEvent)->ViewGroup.dispatcherEvent（viewRootImpl的父类是FrameLayout）->View.onIntercetEvent->onTouch->onTouchEvent->onClick
   >
   > 两种解决滑动冲突的方法：外部拦截：onIntercepterEvent，一般是在父类中判断条件是否需要拦截，如果在down之后的事件拦截，会先给之前处理事件的子view传递cancel事件，然后交给自己的onTouchEvent处理。
   >
   > 内部拦截：requestDisallowIntercepterEvent，父亲中在down事件后onIntercepterEvent设置true，然后子view判断条件设置requestDisallowIntercepterEvent是否运行父view拦截处理。

1. 事件是怎么产生的？mFirstTarget 为什么是一个链表？

   > 内部硬件驱动接收到，然后通过socket传递给wms，wms再传递到viewRootImpl，然后在一级级传递下去，mFirstTarget链表是为了支持多指操作。

1. 自定义View需要经历哪几个过程？

   > 要看场景，一般是先确定自己是继承view，还是viewGroup，确定需要的参数，可以在x m l中配置，然后如果继承view的，一般重新onDraw方法，创建画笔，在画布上绘制。如果viewGroup，那必要重写onlayout，一般也会重写onMeature，在onMeature中需要对子view进行测量，通过子view的测量结果，得到自己的宽高，然后在onlayout中根据测量结果，确定上下左右位置，进行布局。最后进行调试效果。注意在onMeature/onDraw中避免创建对象，因为会经常刷新重绘制，这样会产生各种内存碎片，造成内存抖动，设置内存溢出。

1. A 跳转到 B页面，两个页面的生命周期怎么走？什么情况下A的stop()不会执行。

   > A.onPause->b.onCreate->b.onStart->b.onResume->A.onstop
   >
   > 当b是透明主题时A的stop()不会执行

1. Activity 的4中启动模式分别是什么，有什么不同。

   > 标准模式，就是启动一个，创建一个；
   >
   > singleTop，如果顶部存在，则直接用顶部的，调用onNewIntent，否则，顶部唯一
   >
   > singleTask，如果栈中存在，则会把页面栈上面的页面都结束，然后自己在栈顶调用onNewIntent，栈唯一
   >
   > singleInstance，自己在单独的一个栈中，全局唯一

1. okhttp中有几个队列？分别干什么用的？怎么取消一个请求？

   > 同步队列、异步队列、进行中队列，
   >
   > 同步队列：只是用来做个记录，因为马上会被调用
   >
   > 异步队列，当进行中队列的队列超过64，并且相同域名下的超过5个，则不会执行，等到有任务完成，又会重新从队列中取。
   >
   > 进行中队列，记录正在执行的任务表，

1. Rxjava中map和flatMap有什么区别，都用过什么操作符。

1. 如果Rxjava组合发送任务，中间任务出现异常，其他任务该怎么处理。

1. 哪个场景会发生内存泄露，内存泄露怎么检测，怎么解决。以及leak cannery内部原理是什么？为什么新版本的不需要在Application中注册了。

1. 手机适配问题怎么处理，都有什么方案。

   > 尽量使用weight、linearlayout布局，使用和像素无关的单位ps, dp
   >
   > 头条适配方案
   >
   > 图片尽量放大图在高dpi下面

1. Android9 10 11 都更新了什么新特性，新版本中无法获取IMEI怎么处理。

   > android 10：分区存储
   >
   > Android8.0之后，序列号的获取跟IMEI权限绑定，如果不授权电话权限，同样获取不到序列号
   > Android 10之后，序列号、IMEI 非系统APP获取不到
   > Android 11.0之后，序列号、IMEI MAC 非系统APP获取不到

1. 数据序列话有那俩种方式，Serialization和Parcelable区别，如果持久化需要用哪一个

   > Serialization

1. 组件化怎么分层，各个组件之间怎么通信。

   > 上层是个app壳用与组合多个业务组件
   >
   > 第二层一般是业务组件如订单、主页、购物车等
   >
   > 第三层基础业务组件支付、分享
   >
   > 第四层基础组件日志、网络等
   >
   > arouter通信方案

1. 怎防止程序崩溃，如果已经到了Thread.UncaughtExceptionHandler是否可以让程序继续运行。

   > 

1. Handler Looper mesaageQueue message 之间的关系。

   >  首先是一个线程会只会保存一个Looper，通过 ThreadLocal保障，然后Looper中只有一个messageQueue，looper会开启死循环一直从messageQueue拿message执行，而一个Handler持有一个Looper，通过sendmessageAtTime，往messageQueue中添加message，messageQueue中通过链表的方式存储message，正常情况下message.target只有Handler引用，然后执行message的时候，会执行message.target.dispatcherMessage，也就是执行Handler.dispatcherMessage，里面会先判断message是否有callback，有的话会先执行，然后看handler中是否有callback，有的话先执行，否则执行自己的handlermessage

1. 子线程一定不能更新ui么？什么时候可以？什么时候不可以。检测逻辑是在什么阶段初始化的。

   > 不一定，在onResume及执行都可以，在onResume中会创建viewRootImpl，检查时在viewRootImpl的checkThread方法中。

1. ANR发生的原理是什么， 怎么排查。

   > 输入事件处理：5s
   >
   > Contentprivide 10s
   >
   > 前台广播：10s
   >
   > 后台广播：60s
   >
   > 前台服务：20s
   >
   > 后台服务：200s
   >
   > 
   >
   > 首先可以看log ca t日志，或者查看手机目录下的trace文件，看线程、cpu使用情况，看是哪里执行了耗时操作

1. 程序怎么保活。

1. 说下路由ARoute的实现原理，怎么处理页面过多内存占用过大问题。

   > 用到的技术主要是apt技术，在编译时，每个modoule下都会动态生成路由表文件，还有一个group组文件，然后在应用启动的时候，会去扫描dex文件中指定目录下生成的路由表文件，反射创建，然后讲路由表map传入，调用各个路由边中的方法，放map中添加路由映射，保存方式是先保存group表，然后保存每个group下的路由集，这样在查找的时候，可以先查找group，再查找group下的路由，速度更快。这里第一次扫描到之后，会将路由表文件的路径保存到本地，下次只要版本不变就不会再从新查找。

1. 线程池都什么时候用，怎么创建，构造函数中的参数分别代表什么意思？

   > 有多个任务，或者大任务需要拆分执行，充分利用cpu时，可以使用线程池，一般创建 EextoturPool
   >
   > 核心线程数，但一个任务来的时候，如果线程数小于核心线程数，则会创建核心线程数执行
   >
   > 线程总数，当线程总数超过的时候会放入阻塞队列
   >
   > 阻塞队列：分很多种，当线程总数超过的时候会放入阻塞队列
   >
   > 超时时间：当核心线程数空闲时间超过超时时间时，会回收
   >
   > 创建工厂
   >
   > 拒绝策略：当阻塞队列也满的时候，回调处理

1. 进程优先级

   > 越大优先级越高

1. 反向输出字符串

1. 两个有序链表合并

   > 双指针操作

1. 字符串移除多余空格，且技术单词首字符大写。

    

1. 二叉树中和为某一值的路径

1. 本地广播和正常广播的区别

   > 本地广播

1. 二进制低位转高位

1. 字符串数组判重

1. 二叉树 判断是否为搜索二叉树

1. Activity启动流程，Launcher启动流程











1、livedata.observer，会先把观察者包装下，然后再添加进observers集合中，然后在setvalue中，去分发事件

会先看是不是active状态，





<img src="/Users/liujian/Library/Application Support/typora-user-images/image-20220324110333593.png" alt="image-20220324110333593" style="zoom:50%;" />



技术方案上的选型、架构上的选型、

1、小电flutter版本重构、文档建立，人员配比+热更新不要求+业务长期未维护，文档缺失

2、电小二配置散乱、无统一入口、接口域名散乱、无文档、打包流程

3、扫脸项目，解决手机开不了机的痛点（需求评审、时间评估、项目中任务同步、厂商沟通调试、第三方支付沟通联调、风险评估、上线、线下安装培训、柜机设备绑定问题），只有安卓端，技术选型kotlin+jetpack

> 问题点：接口流程过长、用户扫描到取宝时间过长
>
> 流程：
>
> 检查设备注册->获取绑定设备->获取业务参数->获取本地刷脸设备信息->获取授权信息->发起刷脸请求(获取fToken、alipayUid)->注册登陆->检查设备->.   创建
>
> 解决思路：
>
> 
>
> 

4、短视频项目（充分利用线下5000bd，利好商家，谈合同筹码，短视频平台收益，后期自己短视频平台），技术选型flutter



#### 电小二重构

原先的问题：渠道多、native/h5接口混在一起配置，配置的方式是组合式（url域名拆分，根据不同的渠道、环境去动态修改）、修改点不统一、可以随意修改

修改思路：

1、整理接口：将native/h5接口分开、渠道也分开

2、对与native域名处理（特点、同时存在多套、不同的接口域名不统一、需要动态修改）：设置默认域名，提供不同域名注解（z/o）、通过拦截器识别注解修改，和渠道接藕、只和接口本身和环境相关

3、h5接口（特点、不会更改、渠道确定、域名确定）、根据渠道和环境分发

4、打包接入ci，commit配置识别打包