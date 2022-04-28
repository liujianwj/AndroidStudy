- 请简述一下什么是kotlin，它有什么特性

  > kotlin是静态类型的编程语言，运行于jvm之上，与Java 100%兼容。
  >
  > 特点：
  >
  > 1. time
  > 2. streams
  > 3. try-with-resources
  > 4. 函数扩展，给types、classes或者interfaces新增方法
  > 5. null safe
  > 6. 不需要new，后缀声明类型
  > 7. 自动转换有getters和setters综合属性的类型，例如自动替换getDay()为day，看起来像个field，但实际上是property-getter和setter的概念的融合
  > 8. 函数表达式lambdas，it：单个参数的隐式名称
  > 9. Higher-order函数，一个参数式函数或者返回时函数的函数
  > 10. 扩展函数表达式 = 扩展函数 + 函数表达式 + 高阶函数
  > 11. in-line函数
  > 12. Anko 定义UI

- kotlin中注解@JvmOverloads的作用

  > 在Kotlin中@JvmOverloads注解的作用就是：在有默认参数值的方法中使用@JvmOverloads注解，则Kotlin就会暴露多个重载方法。
  > 在 Kotlin 中调用默认参数值的方法或者构造函数是完全没问题的，但是在 Java 代码调用相应 Kotlin 代码却不行，也就是，Java 代码不能调用在 Kotlin 中使用默认值实现的重载函数或构造函数。
  > @JvmOverloads 就是解决这一问题的，从命名 —— “Jvm 重载”
  >
  > 如果我们再kotlin中写如下代码：
  >
  > ```kotlin
  > fun f(a: String, b: Int = 0, c: String="abc"){}
  > ```
  >
  > 相当于在Java中声明
  >
  > ```java
  > void f(String a, int b, String c){}
  > ```
  >
  >
  > 默认参数没有起到任何作用。
  >
  > 但是如果使用的了@JvmOverloads注解：
  >
  > ```kotlin
  > @JvmOverloads fun f(a: String, b: Int=0, c:String="abc"){}
  > ```
  >
  > 相当于在Java中声明了3个方法：
  >
  > ```java
  > void f(String a)
  > void f(String a, int b)
  > void f(String a, int b, String c)
  > ```
  >
  > 是不是很方便，再也不用写那么多重载方法了。
  >
  > 注：该注解也可用在构造方法和静态方法。
  >
  > ```kotlin
  > class MyLayout: RelativeLayout {
  > 
  >    @JvmOverloads
  >    constructor(context:Context, attributeSet: AttributeSet? = null, defStyleAttr: Int = 0): super(context, attributeSet, defStyleAttr)
  > }
  > ```
  >
  > 
  >
  > 相当Java中的：
  >
  > public class MyLayout extends RelativeLayout {
  >
  > ```java
  > public MyLayout(Context context) {
  >     this(context, null);
  > }
  > 
  > public MyLayout(Context context, AttributeSet attrs) {
  >     this(context, attrs, 0);
  > }
  > 
  > public MyLayout(Context context, AttributeSet attrs, int defStyleAttr) {
  >     super(context, attrs, defStyleAttr);
  > }
  > }
  > ```

- Kotlin中List与MutableList的区别

  > List：有序接口，只能读取，不能更改元素；
  > MutableList：有序接口，可以读写与更改、删除、增加元素。

- Kotlin中实现单例的几种常见方式 https://www.jianshu.com/p/5797b3d0ebd0

  > - 饿汉式
  > - 懒汉式
  > - 线程安全的懒汉式
  > - 双重校验锁式
  > - 静态内部类式
  >
  > 1. 饿汉式（线程安全）
  >
  >    ```kotlin
  >    //Java实现
  >    public class SingletonDemo {
  >        //利用了类加载机制来保证只创建一个instance实例
  >        private static SingletonDemo instance=new SingletonDemo();
  >     private SingletonDemo(){
  >    
  >        }
  >        public static SingletonDemo getInstance(){
  >            return instance;
  >        }
  >    }
  >    //Kotlin实现
  >    object SingletonDemo
  >    ```
  >    
  > 2. 懒汉式（线程不安全）
  >
  >    ```kotlin
  >    //Java实现
  >    public class SingletonDemo {
  >        private static SingletonDemo instance;
  >        private SingletonDemo(){}
  >        public static SingletonDemo getInstance(){
  >            if(instance==null){
  >                instance=new SingletonDemo();
  >            }
  >            return instance;
  >        }
  >    }
  >    //Kotlin实现
  >    class SingletonDemo private constructor() {
  >        companion object {
  >            private var instance: SingletonDemo? = null
  >                get() {
  >                    if (field == null) {
  >                        field = SingletonDemo()
  >                    }
  >                    return field
  >                }
  >            fun get(): SingletonDemo{
  >            //细心的小伙伴肯定发现了，这里不用getInstance作为为方法名，是因为在伴生对象声明时，内部已有getInstance方法，所以只能取其他名字
  >             return instance!!
  >            }
  >        }
  >    }
  >    ```
  >
  > 3. 线程安全的懒汉式
  >
  >    ```java
  >    //Java实现
  >    public class SingletonDemo {
  >        private static SingletonDemo instance;
  >        private SingletonDemo(){}
  >        public static synchronized SingletonDemo getInstance(){//使用同步锁
  >            if(instance==null){
  >                instance=new SingletonDemo();
  >            }
  >            return instance;
  >        }
  >    }
  >    //Kotlin实现
  >    class SingletonDemo private constructor() {
  >        companion object {
  >            private var instance: SingletonDemo? = null
  >                get() {
  >                    if (field == null) {
  >                        field = SingletonDemo()
  >                    }
  >                    return field
  >                }
  >            @Synchronized
  >            fun get(): SingletonDemo{
  >                return instance!!
  >            }
  >        }
  >    
  >    }
  >    ```
  >
  > 4. 双重校验锁式
  >
  >    ```java
  >    //Java实现
  >    public class SingletonDemo {
  >        private volatile static SingletonDemo instance;
  >        private SingletonDemo(){} 
  >        public static SingletonDemo getInstance(){
  >            if(instance==null){
  >                synchronized (SingletonDemo.class){
  >                    if(instance==null){
  >                        instance=new SingletonDemo();
  >                    }
  >                }
  >            }
  >            return instance;
  >        }
  >    }
  >    //kotlin实现
  >    class SingletonDemo private constructor() {
  >        companion object {
  >            val instance: SingletonDemo by lazy(mode = LazyThreadSafetyMode.SYNCHRONIZED) {
  >            SingletonDemo() }
  >        }
  >    }
  >    
  >    //带参数
  >    class SingletonDemo private constructor(private val property: Int) {//这里可以根据实际需求发生改变
  >      
  >        companion object {
  >            @Volatile private var instance: SingletonDemo? = null
  >            fun getInstance(property: Int) =
  >                    instance ?: synchronized(this) {
  >                        instance ?: SingletonDemo(property).also { instance = it }
  >                    }
  >        }
  >    }
  >    ```
  >
  > 5. 静态内部类式
  >
  >    ```kotlin
  >    //Java实现
  >    public class SingletonDemo {
  >      //静态内部类，只有静态内部类的静态属性被调用时，才会加载，可实现延时加载功能。
  >        private static class SingletonHolder{
  >            private static SingletonDemo instance=new SingletonDemo();
  >        }
  >        private SingletonDemo(){
  >            System.out.println("Singleton has loaded");
  >        }
  >        public static SingletonDemo getInstance(){
  >            return SingletonHolder.instance;
  >        }
  >    }
  >    //kotlin实现
  >    class SingletonDemo private constructor() {
  >        companion object {
  >            val instance = SingletonHolder.holder
  >        }
  >    
  >        private object SingletonHolder {
  >            val holder= SingletonDemo()
  >        }
  >    
  >    }
  >    ```
  
- data关键字的理解，相对于普通类有什么特点

  > 在Java中，如果我们定义一个Cellphone数据类，则需要实现其基本的属性和行为，比如：构造函数、equals函数、hashcode函数、toString函数等，相关内容如下:
  >
  > ```java
  > class Cellphone{
  > 	String brand;
  > 	double price;
  > 	
  > 	public Cellphone(String brand, double price){
  > 		this.brand = brand;
  > 		this.price = price;
  > 	}
  > 	
  > 	public boolean equals(Object obj){
  > 		//instanceof严格来说是Java中的一个双目运算符,用来测试一个对象是否为一个类的实例
  > 		//其中obj为一个对象,Class表示一个类或者一个接口
  > 		//当obj为Class的对象,或者是其直接或间接子类,或者是其接口的实现类时
  > 		//结果result都返回true，否则返回false。
  > 		if(obj instanceof Cellphone){
  > 			Cellphone other = (Cellphone)obj;
  > 			return other.brand.equals(brand) && other.price == price;
  > 		}
  > 		return false;
  > 	}
  > 	
  > 	public int hashCode(){
  > 		//根据哈希算法算出来的一个值，这个值跟地址值有关，但不是实际地址值
  > 		return brand.hashCode()+ (int)price;
  > 	}
  > 	
  > 	public String toString(){
  > 		return "Cellphone(brand=" + brand +", price=" + price+")";
  > 	}
  > 
  > ```
  >
  > 注：String类型内容的比较，我们用[equals](https://so.csdn.net/so/search?q=equals&spm=1001.2101.3001.7020)方法，不能用==，==比较的是是否为同一个引用对象（地址）。
  >
  > 但是在Kotlin中，我们只需要使用**data关键字**就可以定义一个数据类，仅仅需要一行代码就可以搞定：
  >
  > ```kotlin
  > data class Cellphone(val brand: String, val price: Double)
  > fun main( ){
  >     val c1 = Cellphone("小米", 3600.0)
  >     val c2 = Cellphone("华为", 3600.0)
  >     println(c1.equals(c2))
  >     println(c1 == c2)  
  >     println(c1 === c2) //kotlin中==比较的是数值是否相等, 而===比较的是两个对象的地址是否相等
  >     println(c1.hashCode())
  >     println(c1.toString())
  > }
  > 
  > //输出
  > true
  > true
  > false
  > 1108657724
  > 1108657724
  > Cellphone(brand=小米, price=3600.0)
  > ```

- 什么是委托属性？场景、原理。https://www.jianshu.com/p/b13d27249959

  > by关键词
  >
  > 1. 接口委托
  >
  >    ```kotlin
  >    // 约束类
  >    interface IGamePlayer {
  >        // 打排位赛
  >        fun rank()
  >        // 升级
  >        fun upgrade()
  >    }
  >    
  >    // 被委托对象，本场景中的游戏代练
  >    class RealGamePlayer(private val name: String): IGamePlayer{
  >        override fun rank() {
  >            println("$name 开始排位赛")
  >        }
  >    
  >        override fun upgrade() {
  >           println("$name 升级了")
  >        }
  >    
  >    }
  >    
  >    // 委托对象，by接被委托对象
  >    class DelegateGamePlayer(private val player: IGamePlayer): IGamePlayer by player
  >    
  >    // Client 场景测试
  >    fun main() {
  >        val realGamePlayer = RealGamePlayer("张三")
  >        val delegateGamePlayer = DelegateGamePlayer(realGamePlayer)
  >        delegateGamePlayer.rank()
  >        delegateGamePlayer.upgrade()
  >    }
  >    
  >    //输出结果
  >    张三 开始排位赛
  >    张三 升级了
  >    ```
  >
  >    内部原理，通过反编译可以看到，其实就是在DelegateGamePlayer中的实现方法中，自动去调用被委托对象的方法。
  >
  > 2. 属性委托
  >
  >    方式一：自己写operator-getValue，setValue方法，参数不太好理解
  >
  >    ```kotlin
  >    class Delegate {
  >        operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
  >            return "$thisRef, thank you for delegating '${property.name}' to me!"
  >        }
  >    
  >        operator fun setValue(thisRef: Any?, property: KProperty<*>, value: String) {
  >            println("$value has been assigned to '${property.name}' in $thisRef.")
  >        }
  >    }
  >    
  >    class Test {
  >        // 属性委托
  >        var prop: String by Delegate()
  >    }
  >    
  >    fun main() {
  >        println(Test().prop)
  >        Test().prop = "Hello, Android技术杂货铺！"
  >    }
  >    
  >    //输出结果
  >    Test@5197848c, thank you for delegating 'prop' to me!
  >    Hello, Android技术杂货铺！ has been assigned to 'prop' in Test@17f052a3.
  >    ```
  >
  >    方法二：实现标准库中声明了2个含所需 `operator`方法的 `ReadOnlyProperty / ReadWriteProperty` 接口
  >
  >    ```kotlin
  >    // val 属性委托实现
  >    class Delegate1: ReadOnlyProperty<Any,String>{
  >        override fun getValue(thisRef: Any, property: KProperty<*>): String {
  >            return "通过实现ReadOnlyProperty实现，name:${property.name}"
  >        }
  >    }
  >    // var 属性委托实现
  >    class Delegate2: ReadWriteProperty<Any,Int>{
  >        override fun getValue(thisRef: Any, property: KProperty<*>): Int {
  >            return  20
  >        }
  >    
  >        override fun setValue(thisRef: Any, property: KProperty<*>, value: Int) {
  >           println("委托属性为： ${property.name} 委托值为： $value")
  >        }
  >    
  >    }
  >    // 测试
  >    class Test {
  >        // 属性委托
  >        val d1: String by Delegate1()
  >        var d2: Int by Delegate2()
  >    }
  >    
  >    fun main(){
  >      val test = Test()
  >        println(test.d1)
  >        println(test.d2)
  >        test.d2 = 100
  >    }
  >    
  >    //输出结果
  >    通过实现ReadOnlyProperty实现，name:d1
  >    20
  >    委托属性为： d2 委托值为： 100
  >    ```
  >
  >    通过反编译可以发现在Test构造方法中会自动创建Delegate1对象，并且会在Test中新增getD1()，setD1()方法，且在方法中调用Delegate1对象的getValue，setValue方法。
  >
  > 3. kotlin标准库中提供的委托
  >
  >    - 延迟属性（lazy properties）: 其值只在首次访问时计算；
  >
  >    - 可观察属性（observable properties）: 监听器会收到有关此属性变更的通知；
  >
  >    - 把多个属性储存在一个映射（map）中，而不是每个存在单独的字段中。

- with、apply场景和区别

- Unit类型和java中的void的区别

  > Kotlin中的 Unit 类型实现了与Java中的 void 一样的功能。不同的是，当一个函数没有返回值的时候，我们用 Unit 来表示这个特征，而不是 null 。
  >
  > Unit、Any、Nothing
  >
  > https://www.jianshu.com/p/31e777c64620
  >
  > 
  >
  > `Nothing` 是最难理解的一个概念，不同于 `Unit` 至于 `Void` ，`Any` 之于 `Object` ， `Nothing` 在Java中并没有一个相似的概念。
  >
  > 当函数中死循环，或者抛出异常，即函数永远没有返回值时，返回`Nothing`

- infix关键字的原理和场景

  > 高级语法糖，增加可读性，创建map键值对时就时使用这个，让A.to(B)可以写成A to B
  >
  > ```kotlin
  > //正常的startsWith
  > if("Hello Wonderful".startsWith("Hello")) {
  >     // to do something
  > }
  > //infix修饰后
  > infix fun String.beginsWith(prefix: String) = startsWith(prefix)
  > 
  > //修饰后的使用
  > if ("Hello Wonderful" beginsWith "Hello"){
  >     // to do something
  > }
  > ```

- 可见修饰符有哪些？相比于java区别

  > - private（与java一样）
  > - protected（与java一样）
  > - internal. (对应java的default，模块可见，，java是包可见，java中默认)
  > - public（kotlin中默认，作用范围与java一样）

- 与java混合开发时需要注意什么

- 何为解构，如何使用  https://www.jianshu.com/p/61cb749a26d3

  > ```kotlin
  > data class Book(var name: String, var price: Float)
  > 
  > fun main(args: Array<String>) {
  >     //将Book解构
  >     val (name, price) = Book("Kotlin入门", 66.6f)
  >     println(name)
  >     println(price)
  > }
  > // 输出
  > Kotlin入门
  > 66.6
  > 
  > for(int i = 0; i < weight.size(); i++) { // 遍历物品
  >     for(int j = bagWeight; j >= weight[i]; j--) { // 遍历背包容量
  >         dp[j] = max(dp[j], dp[j - weight[i]] + value[i]);
  > 
  >     }
  > }
  > 
  > // 开始 01背包
  > for(int i = 0; i < nums.size(); i++) {
  >     for(int j = target; j >= nums[i]; j--) { // 每一个元素一定是不可重复放入，所以从大到小遍历
  >         dp[j] = max(dp[j], dp[j - nums[i]] + nums[i]);
  >     }
  > }
  > 
  > 套到本题，dp[j]表示 背包总容量是j，最大可以凑成j的子集总和为dp[j]。
  > 
  > 
  > 给定一个非负整数数组，a1, a2, ..., an, 和一个目标数，S。现在你有两个符号 + 和 -。对于数组中的任意一个整数，你都可以从 + 或 -中选择一个符号添加在前面。
  > 
  > 返回可以使最终数组和为目标数 S 的所有添加符号的方法数。
  > 
  > 示例：
  > 
  > 输入：nums: [1, 1, 1, 1, 1], S: 3
  > 输出：5
  > 
  > 解释：
  > -1+1+1+1+1 = 3
  > +1-1+1+1+1 = 3
  > +1+1-1+1+1 = 3
  > +1+1+1-1+1 = 3
  > +1+1+1+1-1 = 3
  > 
  > 一共有5种方法让最终目标和为3。
  > 
  > 提示：
  > 
  > 数组非空，且长度不会超过 20 。
  > 初始的数组的和不会超过 1000 。
  > 保证返回的最终结果能被 32 位整数存下。
  > #
  > ```
  >
  > 

- inline 什么时内联函数，什么作用，reified

  > 被inline修饰得方法，是内联函数
  >
  > 在反编译中发现，调用内联函数时，其实会把该函数体中的内容都搬过来，这样减少了方法入栈和出栈的消耗
  >
  > reified，在内联函数中修饰泛型，可以获取实例的class类型。

- 构造方法，注意事项

  > https://www.kotlincn.net/docs/reference/classes.html

- sequence，为什么它处理集合操作更高效 https://blog.csdn.net/u013064109/article/details/84981570

  > 当最后的操作是获取first 或者last时，大数据且有多个操作时，使用sequence才更加有效
  >
  > 原因是因为sequence内部使用一个迭代器，比如我们先map，然后filter，最后first，，sequence在处理一个item的Map操作后，会直接给filter处理，最后给first，然后再处理下一个item。这样我们一获取到first符合结果值时就直接返回了。且多个操作时，一次遍历就能完成操作，list需要买次操作时都遍历一遍。
  >
  > 原理：https://juejin.cn/post/6844903667947012110

- coroutines，与线程的区别，哪些优点

- 如何安全的处理可空类型

- Any与java中的Object区别

- 数据类型有隐式转换吗，为什么

- 集合遍历有哪几种方式