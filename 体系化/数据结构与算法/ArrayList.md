#### ArrayList 几个重要属性之一

1. 首先是默认初始值的大小：

   ```java
   java private static final int DEFAULT_CAPACITY = 10;
   ```

2. 接着是一个默认的空对象数组：

   ```java
   private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
   ```

3. 然后是ArrayList 实际数据存储的一个数组：

   ```java
   transient Object[] elementData;
   ```

4. elementData 的大小：

   ```java
   private int size;
   ```

   

#### 来个Demo，看一下ArrayList 的使用
1. 首先来个辅助方法，用于获得ArrayList 的容量大小，由于即elementData 数据的长度，由于elementData 是私有的属性，所以，我们可以用反射的机制来进行获取。如下。

   ```java
    public static Integer getCapacity(ArrayList list) {
        Integer length = null;
        Class c = list.getClass();
        Field f;
        try {
            f = c.getDeclaredField("elementData");
            f.setAccessible(true);
            Object[] o = (Object[]) f.get(list);
            length = o.length;
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
        return  length;
    }
   ```

2. 看一下，如果构造ArrayList 对象的时候，如果在构造函数里面什么也不传，其会进行什么操作。可以看到，其进行的操作是，初始化了一个空的数组。

```java
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}        
```
3. 接下来我们调用无参构造函数，来创建一个对象。看一下其初始化容量是，以及大小是多少。最终的结果：容量：0，大小：0，

   ```java
    ArrayList<String> list = new ArrayList<>();
    Integer length = getCapacity(list);
    int size = list.size();
    System.out.println("容量： " + length);
    System.out.println("大小： " + size);
   ```

4. 接着，我们构造一个ArrayList，添加一个元素，看一下其容量。最终的结果，容量：10，大小：1，可以看出。也就是说如果ArrayList 构造函数中如果没有设置初始化的容量大小，在没有添加有元素的时候，其初始化容量是0，只有当添加第一个元素的时候，才会初始化容量才会设置成10.

```java
    ArrayList<String> list = new ArrayList<>();
    Integer length = getCapacity(list);
    int size = list.size();
    System.out.println("容量： " + length);
    System.out.println("大小： " + size);
```
#### ArrayList 扩容机制
1. 接下来，我们进行构造一个ArrayList 对象，同样调用的是无参构造函数。然后往里面添加11个对象，看起容量会如何进行动态扩展。最终结果，容量：15，大小：11

```java
    ArrayList<String> list = new ArrayList<>();
    for (int i = 1; i <= 11; i++) {
        list.add("value" + i);
    }
    Integer length = getCapacity(list);
    int size = list.size();
    System.out.println("容量： " + length);
    System.out.println("大小： " + size);
```
2. 要清楚其扩容机制，我们需要跟进去，看一下其add 方法，

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```

首先肯定是得确保数组的容量够到，这样才不会包数组越界异常，IndexOutOfBoundsException。

3. 看一下ensureCapacityInternal方法的实现：

   ```java
   private void ensureCapacityInternal(int minCapacity) {
       if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
           minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
       }
       ensureExplicitCapacity(minCapacity);
   }
   private void ensureExplicitCapacity(int minCapacity) {
       modCount++;
       // overflow-conscious code
       if (minCapacity - elementData.length > 0)    // 如果其元素个数大于其容量，则进行扩容。
           grow(minCapacity);    
   }
   ```

   

4. 具体扩容流程：

```java
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;   // 原来的容量
    int newCapacity = oldCapacity + (oldCapacity >> 1);  // 新的容量，原来容量的1.5倍。
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)  // 如果大于ArrayList 可以容许的最大容量，则设置为最大容量。
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);  // 最终利用Arrays.coppy 进行扩容，生成一个1.5倍元素的数组。（即例子中的15个元素的数组。）
}
```

#### 最后来个总结：
ArrayList 的内部实现，其实是用一个对象数组进行存放具体的值，然后用一种扩容的机制，进行数组的动态增长。

其扩容机制可以理解为，如果元素的个数，大于其容量，则把其容量扩展为原来容量的1.5倍。


#### 线程安全问题

1. 使用 Vector 初始化 list 对象

```java
List<String> list = new Vector<>();
```

因为Vector.add使用了synchronized加锁

```java
    public synchronized boolean add(E e) {
        modCount++;
        ensureCapacityHelper(elementCount + 1);
        elementData[elementCount++] = e;
        return true;
    }
```

2. 使用 Collections.synchronizedList

```java
List<String> list = Collections.synchronizedList(new ArrayList<>());
```

这种情况下，调用的是SynchronizedList.add，源码如下，同样做了加锁

```java
public void add(int index, E element) {
    synchronized (mutex) {list.add(index, element);}
}
```

3. 使用 CopyOnWriteArrayList

```java
List<String> list = new CopyOnWriteArrayList<>();
```

底层使用的是ReentrantLock，源码如下：

```java
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```