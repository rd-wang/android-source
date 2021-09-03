本文所用源码来自android-29/java/util/HashMap.java
# HashMap
## 哈希表
什么是哈希
翻译成 “散列” ，就是把任意长度的输入，通过散列算法，变成固定长度的输出，该输出就是散列值，这个映射函数叫做散列函数，存放记录的数组叫做散列表。

数据结构的物理存储结构只有两种：**顺序存储结构**和**链式存储结构**（像栈，队列，树，图等是从逻辑结构去抽象的，映射到内存中，也这两种物理组织形式），在数组中根据下标查找某个元素，一次定位就可以达到，哈希表利用了这种特性，**哈希表的主干就是数组。** 新增或查找某个元素，我们通过把当前元素的关键字 通过某个函数映射到数组中的某个位置，通过数组下标一次定位就可完成操作。
$$
存储位置 = f(关键字)
$$
这个函数f一般称为哈希函数，这个函数的设计好坏会直接影响到哈希表的优劣。


>  数组：采用一段连续的存储单元来存储数据。对于指定下标的查找，时间复杂度为O(1)；通过给定值进行查找，需要遍历数组，逐一比对给定关键字和数组元素，时间复杂度为O(n)，当然，对于有序数组，则可采用二分查找，插值查找，斐波那契查找等方式，可将查找复杂度提高为O(logn)；对于一般的插入删除操作，涉及到数组元素的移动，其平均复杂度也为O(n)

## 哈希冲突
如果**两个不同的元素，通过哈希函数得出的实际存储地址相同**怎么办？也就是说，当我们对某个元素进行哈希运算，得到一个存储地址，然后要进行插入的时候，发现已经被其他元素占用了，其实这就是所谓的**哈希冲突**，也叫**哈希碰撞**。


哈希函数的设计至关重要，好的哈希函数会尽可能地保证 计算简单和散列地址分布均匀，再好的哈希函数也不能保证得到的存储地址绝对不发生冲突。那么哈希冲突如何解决呢？哈希冲突的解决方案有多种:开放定址法（发生冲突，继续寻找下一块未被占用的存储地址），再散列函数法，链地址法，而**HashMap即是采用了链地址法**，也就是数组+链表的方式。

## HashMap的类继承关系

<div align=center><img src="https://img-blog.csdnimg.cn/3581e8cf20ae4552995046aff1ecea09.jpg"  width= 70%>

下面针对各个实现类的特点做一些说明：

1. HashMap：它根据键的hashCode值存储数据，大多数情况下可以直接定位到它的值，因而具有很快的访问速度，但遍历顺序却是不确定的。 HashMap最多只允许一条记录的键为null，允许多条记录的值为null。HashMap非线程安全，即任一时刻可以有多个线程同时写HashMap，可能会导致数据的不一致。如果需要满足线程安全，可以用 Collections的synchronizedMap方法使HashMap具有线程安全的能力，或者使用ConcurrentHashMap。

2. Hashtable：Hashtable是遗留类，很多映射的常用功能与HashMap类似，不同的是它承自Dictionary类，并且是线程安全的，任一时间只有一个线程能写Hashtable，并发性不如ConcurrentHashMap，因为ConcurrentHashMap引入了分段锁。Hashtable不建议在新代码中使用，不需要线程安全的场合可以用HashMap替换，需要线程安全的场合可以用ConcurrentHashMap替换。

3. LinkedHashMap：LinkedHashMap是HashMap的一个子类，保存了记录的插入顺序，在用Iterator遍历LinkedHashMap时，先得到的记录肯定是先插入的，也可以在构造时带参数，按照访问次序排序。

4. TreeMap：TreeMap实现SortedMap接口，能够把它保存的记录根据键排序，默认是按键值的升序排序，也可以指定排序的比较器，当用Iterator遍历TreeMap时，得到的记录是排过序的。如果使用排序的映射，建议使用TreeMap。在使用TreeMap时，key必须实现Comparable接口或者在构造TreeMap传入自定义的Comparator，否则会在运行时抛出java.lang.ClassCastException类型的异常。

对于上述四种Map类型的类，要求映射中的key是不可变对象。
## HashMap 总体介绍
从结构实现来讲，HashMap是数组+链表+红黑树（JDK1.8增加了红黑树部分）

## HashMap 存储结构

HashMap由数组+链表+红黑树组成的，数组是HashMap的主体，链表则是主要为了解决哈希冲突而存在的，而当链表长度太长（默认超过8）时，链表就转换为红黑树，利用红黑树快速增删改查的特点提高HashMap的性能，其中会用到红黑树的插入、删除、查找等算法，如果定位到的数组位置不含链表（当前entry的next指向null）,那么查找，添加等操作很快，仅需一次寻址即可；如果定位到的数组包含链表，对于添加操作，其时间复杂度为O(n)，首先遍历链表，存在即覆盖，否则新增；对于查找操作来讲，仍需遍历链表，然后通过key对象的equals方法逐一比对查找。所以，性能考虑，HashMap中的链表出现越少，性能才会越好。
>想了解更多红黑树数据结构的工作原理可以参考[http://blog.csdn.net/v_july_v/article/details/6105630](http://blog.csdn.net/v_july_v/article/details/6105630)。

### 整体存储结构

<div align=center><img src="https://img-blog.csdnimg.cn/3643c052d6bf408f950a3a71b7d0e4ca.png"  width= 120%>

成员变量 table 就是哈希桶数组
```java
    /**
     * The table, initialized on first use, and resized as
     * necessary. When allocated, length is always a power of two.
     * (We also tolerate length zero in some operations to allow
     * bootstrapping mechanics that are currently not needed.)
     */
    transient Node<K, V>[] table;
```
Node是HashMap的一个内部类，实现了Map.Entry接口，本质是就是一个映射(键值对)。
```java
/**
     * Basic hash bin node, used for most entries.  (See below for
     * TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
     */
    static class Node<K, V> implements Map.Entry<K, V> {
        final int hash;
        final K key;
        V value;
        Node<K, V> next;

        Node(int hash, K key, V value, Node<K, V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
        public final K getKey() {return key;}
        public final V getValue() {return value;}
        public final String toString() {return key + "=" + value;}
        public final int hashCode(){returnObjects.hashCode(key)^Objects.hashCode(value);}
        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }
        public final boolean equals(Object o) {
            if (o == this)return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?, ?> e = (Map.Entry<?, ?>) o;
                if (Objects.equals(key, e.getKey()) &&
                        Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }
```

### 其他重要成员变量
size 实际存储的key-value键值对的个数
```java
	/**
     * The number of key-value mappings contained in this map.
     */
    transient int size;
```

loadFactor 负载因子，代表了table的填充度有多少，默认是0.75,也就是说大小为16的HashMap，到了第13个元素，就会扩容成32。
>默认的负载因子0.75是对空间和时间效率的一个平衡选择，建议大家不要修改，除非在时间和空间比较特殊的情况下，如果内存空间很多而又对时间效率要求很高，可以降低负载因子Load factor的值；相反，如果内存空间紧张而对时间效率要求不高，可以增加负载因子loadFactor的值，这个值可以大于1。
```java
    /**
     * The load factor for the hash table.
     *
     * @serial
     */
    final float loadFactor;
```

threshold 阈值，
当table == null时，该值为初始容量（初始容量默认为16）
当table != null，也就是为table分配内存空间后，

threshold一般为 capacity*loadFactory

>结合负载因子的定义公式可知，threshold就是在此loadFactor和capacity(数组长度)对应下允许的最大元素数目，超过这个数目就重新resize(扩容)，扩容后的HashMap容量是之前容量的两倍。

```java
  /**
     * The next size value at which to resize (capacity * load factor).
     *
     * @serial
     */
    // (The javadoc description is true upon serialization.
    // Additionally, if the table array has not been allocated, this
    // field holds the initial array capacity, or zero signifying
    // DEFAULT_INITIAL_CAPACITY.)
    int threshold;
```


modCount  HashMap被改变的次数，由于HashMap非线程安全，在对HashMap进行迭代时，
如果期间其他线程的参与导致HashMap的结构发生变化了（比如put，remove等操作），
需要抛出异常ConcurrentModificationException
```java
    /**
     * The number of times this HashMap has been structurally modified
     * Structural modifications are those that change the number of mappings in
     * the HashMap or otherwise modify its internal structure (e.g.,
     * rehash).  This field is used to make iterators on Collection-views of
     * the HashMap fail-fast.  (See ConcurrentModificationException).
     */
    transient int modCount;
```

## HashMap的构造函数
HashMap有4个构造函数

```java
    /**
     * Constructs an empty <tt>HashMap</tt> with the specified initial
     * capacity and load factor.
     *
     * @param initialCapacity the initial capacity
     * @param loadFactor      the load factor
     * @throws IllegalArgumentException if the initial capacity is negative
     *                                  or the load factor is nonpositive
     */
    public HashMap(int initialCapacity, float loadFactor) {
    	// 此处对传入的初始容量进行校验，容量应处于0到MAXIMUM_CAPACITY = 1<<30之间
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                    initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                    loadFactor);
        this.loadFactor = loadFactor;
        //如果输入了初始容量 则会返回最接近 所能容纳下输入容量 的2的n次幂的值
        //比如输入了 30 则会返回32，输入10则会返回16
        this.threshold = tableSizeFor(initialCapacity);
    }
     /**
     * Returns a power of two size for the given target capacity.
     */
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }

    /**
     * Constructs an empty <tt>HashMap</tt> with the specified initial
     * capacity and the default load factor (0.75).
     *
     * @param initialCapacity the initial capacity.
     * @throws IllegalArgumentException if the initial capacity is negative.
     */
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    /**
     * Constructs an empty <tt>HashMap</tt> with the default initial capacity
     * (16) and the default load factor (0.75).
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

   
```
其他构造器如果用户没有传入initialCapacity 和loadFactor这两个参数，会使用默认值
initialCapacity默认为16，loadFactory默认为0.75

在常规构造器中，没有为数组table分配内存空间，是在执行put操作的时候才真正构建table数组
但是有一个入参为指定Map的构造器有些特殊，当传入的map的size大于0时，会计算threshold
但是也没有为数组table分配内存空间，**所以 HashMap 是在执行put操作的时候才真正构建table数组**

```java
	/**
     * Constructs a new <tt>HashMap</tt> with the same mappings as the
     * specified <tt>Map</tt>.  The <tt>HashMap</tt> is created with
     * default load factor (0.75) and an initial capacity sufficient to
     * hold the mappings in the specified <tt>Map</tt>.
     *
     * @param m the map whose mappings are to be placed in this map
     * @throws NullPointerException if the specified map is null
     */
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
    /**
     * Implements Map.putAll and Map constructor
     *
     * @param m     the map
     * @param evict false when initially constructing this map, else
     *              true (relayed to method afterNodeInsertion).
     */
    final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        int s = m.size();
        if (s > 0) {
            if (table == null) { // pre-size
                float ft = ((float) s / loadFactor) + 1.0F;
                int t = ((ft < (float) MAXIMUM_CAPACITY) ?
                        (int) ft : MAXIMUM_CAPACITY);
                if (t > threshold)
                    threshold = tableSizeFor(t);
            } else if (s > threshold)
                resize();
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
            }
        }
    }

```

## HashMap hash()
Hash算法本质上就是三步：取key的hashCode值、高位运算、取模运算。

### 根据key获取哈希桶数组索引位置

```java
/*
此为源码中的注释，看不明白没关系 看下方的文字描述
计算key.hashCode()并将较高的哈希位扩展到较低的哈希位。
由于HashMap使用2的幂掩码，只在当前掩码上方比特上变化的散列集总是会发生冲突。
(在已知的例子中，有一组Float键在小表中保存连续的整数。)
因此，应用了一种转换，将更高位的影响向下传播。在比特传播的速度、效用和质量之间存在权衡。
因为许多常见的哈希集已经合理地分布了(所以不能从分散中获益)，
而且因为使用树来处理箱子中的大量冲突，
所以只是以最便宜的方式对一些移位的位进行异或，
以减少系统损失，同时还考虑了最高位的影响，
否则由于表的边界，这些最高位将永远不会在索引计算中使用。
*/
static final int hash(Object key) {   //jdk1.8 & jdk1.7
     int h;
     // h = key.hashCode() 为第一步 取hashCode值
     // h ^ (h >>> 16)  为第二步 高位参与运算
     return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
static int indexFor(int h, int length) {  //jdk1.7的源码，jdk1.8没有这个方法(减少调用，提高效率)，但是实现原理一样的
     return h & (length-1);  //第三步 取模运算
}
//是一个本地方法，是获取hashCode的值
public native int hashCode();
```
对于任意给定的对象，只要它的hashCode()返回值相同，那么程序调用方法一所计算得到的Hash码值总是相同的。我们首先想到的就是把hash值对数组长度取模运算，这样一来，元素的分布相对来说是比较均匀的。但是，模运算的消耗还是比较大的，在HashMap中是这样做的：  调用 indexFor()

>这个方法非常巧妙，它通过h & (table.length -1)来得到该对象的保存位，而HashMap底层数组的长度总是2的n次方，这是HashMap在速度上的优化。当length总是2的n次方时，h& (length-1)运算等价于对length取模，也就是h%length，但是&比%具有更高的效率。

假设长度是 16 即 10000，也就是说不管怎么说，最高位都是1其余的都是0，
然后n-1 就是最高位是 0 其余都是 1即01111，
那么这个时候 n-1 的范围就是 0-15，
因为数组的下标是从0开始的，所以不管hash是什么值，最后的结果一定是在数组的长度范围之内
对于一个HashMap 他的大小n是固定的，那么hash()的值 直接决定了该元素在数组中的下标。

### HashMap如何处理哈希冲突的问题

那么如何降低哈希冲突，让元素存储的更均匀就是要处理的关键问题那么HashMap是怎么降低hash冲突呢

```java
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

在JDK1.8的实现中，优化了高位运算的算法，通过hashCode()的高16位异或低16位实现的：(h = k.hashCode()) ^ (h >>> 16)，主要是从速度、功效、质量来考虑的，这么做可以在数组table的length比较小的时候，也能保证考虑到高低Bit都参与到Hash的计算中，同时不会有太大的开销。


>异或：相同返回 0 ，不同返回 1

重点来了：
**return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);**  
hash()的值为 h 与 h 经过的无符号右移 16 位的的结果做异或运算

目的是为了让 key 的 hash 值的高 16 位也参与运算。也就是让高 16 位和低 16 位都参与运算，降低hash冲突概率。换句话说：
HashCode 是 int值，32个 bit，如果直接用原始的 HashCode 计算的话：(n - 1) & hash，正常 HashMap 的 size 不会太大，高 16 位参与不到计算位置的运算里，所以计算hash 的时候进行了高 16 位和低 16 位的异或运算，根本目的是为了散列更均匀。

下面举例说明下，n为table的长度。
<div align=center><img src="https://img-blog.csdnimg.cn/9def98f0abd94f9798b1aa242d21d832.jpg"  width= 60%>


## put 源码解析
### put流程图
![在这里插入图片描述](https://img-blog.csdnimg.cn/02a50834480c4b778b82d9938d0683df.webp?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA6Zi_5ouJ6Zi_5Lyv,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)
### put流程描述
①.判断键值对数组table[i]是否为空或为null，否则执行resize()进行扩容；

②.根据键值key计算hash值得到插入的数组索引i，如果table[i]==null，直接新建节点添加，转向⑥，如果table[i]不为空，转向③；

③.判断table[i]的首个元素是否和key一样，如果相同直接覆盖value，否则转向④，这里的相同指的是hashCode以及equals；

④.判断table[i] 是否为treeNode，即table[i] 是否是红黑树，如果是红黑树，则直接在树中插入键值对，否则转向⑤；

⑤.遍历table[i]，判断链表长度是否大于8，大于8的话把链表转换为红黑树，在红黑树中执行插入操作，否则进行链表的插入操作；遍历过程中若发现key已经存在直接覆盖value即可；

⑥.插入成功后，判断实际存在的键值对数量size是否超多了最大容量threshold，如果超过，进行扩容。

### put 源码解析

```java
    //调用putVal()
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    /**
     * Implements Map.put and related methods.
     *
     * @param hash         key 的 hash 值
     * @param key          key 值
     * @param value        value 值
     * @param onlyIfAbsent true：如果某个 key 已经存在那么就不插了；false 存在则替换，没有则新增。这里为 false
     * @param evict        如果为false，表处于创建模式。
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        // tab 表示当前 hash 散列表的引用
        Node<K, V>[] tab;
        // 表示具体的散列表中的元素
        Node<K, V> p;
        // n：表示散列表数组的长度
        // i：表示路由寻址的结果
        int n, i;
        // 将 table 赋值发给 tab ；如果 tab == null，说明 table 还没有被初始化。则此时是需要去创建 table 的
        // 为什么这个时候才去创建散列表？因为可能创建了 HashMap 时候可能并没有存放数据，如果在初始化 HashMap 的时候就创建散列表，势必会造成空间的浪费
        // 这里也就是延迟初始化的逻辑
        if ((tab = table) == null || (n = tab.length) == 0) {
            n = (tab = resize()).length;
        }
        // 如果 p == null，说明寻址到的桶的位置没有元素。那么就将 key-value 封装到 Node 中，并放到寻址到的下标为 i 的位置
        if ((p = tab[i = (n - 1) & hash]) == null) {
            tab[i] = newNode(hash, key, value, null);
        }
        // 到这里说明 该位置已经有数据了，且此时可能是链表结构，也可能是树结构
        else {
            // e 表示找到了一个与当前要插入的key value 一致的元素
            Node<K, V> e;
            // 临时的 key
            K k;
            // p 的值就是上一步 if 中的结果即：此时的 (p = tab[i = (n - 1) & hash]) 不等于 null
            // p 是原来的已经在 i 位置的元素，且新插入的 key 是等于 p中的key
            //说明找到了和当前需要插入的元素相同的元素（其实就是需要替换而已）
            if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
                //将 p 的值赋值给 e
                e = p;
                //说明已经树化，红黑树会有单独的文章介绍，本文不再赘述，不然文章要非常非常的长
            else if (p instanceof TreeNode) {
                e = ((TreeNode<K, V>) p).putTreeVal(this, tab, hash, key, value);
            } else {
                //到这里说明不是树结构，也不相等，那说明不是同一个元素，那就是链表了
                for (int binCount = 0; ; ++binCount) {
                    //如果 p.next == null 说明 p 是最后一个元素，说明，该元素在链表中也没有重复的，那么就需要添加到链表的尾部
                    if ((e = p.next) == null) {
                        //直接将 key-value 封装到 Node 中并且添加到 p的后面
                        p.next = newNode(hash, key, value, null);
                        // 当元素已经是 7了，再来一个就是 8 个了，那么就需要进行树化
                        if (binCount >= TREEIFY_THRESHOLD - 1) {
                            treeifyBin(tab, hash);
                        }
                        break;
                    }
                    //在链表中找到了某个和当前元素一样的元素，即需要做替换操作了。
                    if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k)))) {
                        break;
                    }
                    //将e(即p.next)赋值为e，这就是为了继续遍历链表的下一个元素
                    p = e;
                }
            }
            //如果条件成立，说明找到了需要替换的数据，
            if (e != null) {
                //使用新的值赋值为旧的值
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null) {
                    e.value = value;
                }
                //这个方法没用，里面啥也没有
                afterNodeAccess(e);
                //HashMap put 方法的返回值是原来位置的元素值
                return oldValue;
            }
        }
        // 上面说过，对于散列表的 结构修改次数，那么就修改 modCount 的次数
        ++modCount;
        //size 即散列表中的元素的个数，添加后需要自增，如果自增后的值大于扩容的阈值，那么就触发扩容操作
        if (++size > threshold) {
            resize();
        }
        //啥也没干
        afterNodeInsertion(evict);
        //原来位置没有值，那么就返回 null
        return null;
    }
```

## 扩容机制
假设现在散列表中的元素已经很多了，但是现在散列表的链化已经比较严重了，哪怕是树化了，之间复杂度也没有O(1)好，所以需要扩容来降低Hash冲突的概率，以此来提高性能

扩容(resize)就是重新计算容量，向HashMap对象里不停的添加元素，而HashMap对象内部的数组无法装载更多的元素时，对象就需要扩大数组的长度，以便能装入更多的元素。当然Java里的数组是无法自动扩容的，方法是使用一个新的数组代替已有的容量小的数组。
### 扩容 resize()

```java
    /**
     * Initializes or doubles table size.  If null, allocates in
     * accord with initial capacity target held in field threshold.
     * Otherwise, because we are using power-of-two expansion, the
     * elements from each bin must either stay at same index, or move
     * with a power of two offset in the new table.
     *
     * @return the table
     * 为什么需要扩容？
     * 假设现在散列表中的元素已经很多了，
     * 但是现在散列表的链化已经比较严重了，哪怕是树化了，之间复杂度也没有O(1)好，
     * 所以需要扩容来降低Hash冲突的概率，以此来提高性能
     */
    final Node<K, V>[] resize() {
        // oldTab 表示引用扩容前的 散列表
        Node<K, V>[] oldTab = table;
        // oldCap 扩容前的 table 数组的长度，后面就是一个简单的三目运算符：oldTab 为 null，长度则为 0 ，否则就取 table 实际的长度
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        //表示扩容之前的扩容阈值，也即触发本次 扩容的阈值
        int oldThr = threshold;
        // newCap：扩容之后的 table 的数组的长度
        // newThr：扩容之后下次触发扩容的阈值
        int newCap, newThr = 0;
        //条件成立：说明散列表已经初始化过了，就是一次正常的容量不够了的扩容（因为在 table 没有初始化也会进行 resize 的）
        if (oldCap > 0) {
            //基本的容量大小判断，基本是不可能达到这个数值的，但是为了保持程序的健壮性，还是需要做该检查的。
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                //直接返回原来的容量，已经已经达到最大值，无法再继续扩容了。
                return oldTab;
            }
            //走到这里，首先将 newCap 扩大为原来的 2 倍，且需要判断是否超过了最大值
            //并且要保证扩容之后的容量是大于扩容之前的阈值（16）
            //oldCap >= DEFAULT_INITIAL_CAPACITY 这个条件会不成立吗？假设你创建HashMap 的时候传的初始容量为3 那么就不走这部进行扩容了
            //两个条件都满足以后，那么就将扩容的阈值翻倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
                //将原来的扩容阈值扩大一倍后赋值给新的扩容阈值
                newThr = oldThr << 1;
        }
        // 到这一步说明 oldCap == 0，说明此时散列表中没有任何的元素。但是为什么扩容阈值会可能有大于 0 的情况。
        //需要回头看下构造方法，除了无参构造，别的方法里面最终执行 tableSizeFor()方法。这就导致了 threshold 可能是 > 0 的
        else if (oldThr > 0) {
            newCap = oldThr;
        } else {
            // 到这一步说明 oldTab = 0,oldThr = 0；此时直接非 容量赋值初始值
            newCap = DEFAULT_INITIAL_CAPACITY;
            //通过 容量 * 负载因子 得到 扩容阈值
            newThr = (int) (DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        //这个是什么情况？ 第一种是上面的
        // else if (oldThr > 0) {  newCap = oldThr;  }
        // 的情况下，
        // 还有一种是上面的第一个 if 中的else if 条件没有满足。这个时候 newThr == 0 是成立的
        if (newThr == 0) {
            // 这里面就是在计算新的扩容阈值。
            float ft = (float) newCap * loadFactor;
            //这里真没什么好说的，就是简单的三目运算
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float) MAXIMUM_CAPACITY ? (int) ft : Integer.MAX_VALUE);
        }
        //将新的扩容阈值赋值给 threshold
        threshold = newThr;
        @SuppressWarnings({"rawtypes", "unchecked"})
        //创建一个容量更大的数组
        Node<K, V>[] newTab = (Node<K, V>[]) new Node[newCap];
        //将新数组赋值给 table
        table = newTab;
        //条件成立，说明 原来的散列表中有元素呗
        if (oldTab != null) {
            //扩容没有捷径，就是每个桶位置去处理
            for (int j = 0; j < oldCap; ++j) {
                //e：表示当前 node 节点
                Node<K, V> e;
                //将 j位置的元素赋值给 e，且如果 j 位置元素不为null。否则继续下一轮循环
                if ((e = oldTab[j]) != null) {
                    //将 j 位置置为 null，方便 GC
                    oldTab[j] = null;
                    //如果 e.next 为空，说明该位置没有发生过 hash 碰撞。
                    if (e.next == null) {
                        //计算新的桶的小标，并将e设置进去
                        newTab[e.hash & (newCap - 1)] = e;
                    } else if (e instanceof TreeNode) {
                        //判断是否已经树化，本文不讨论，过~
                        ((TreeNode<K, V>) e).split(this, newTab, j, oldCap);
                    } else {
                        //★★★★★最重要的的地方★★★★★ 处理链表
                        // 低位链表：存放扩容之后的数组的下标位置，与当前数组的下标位置是一致的（/image/hashmap/图来解释）
                        Node<K, V> loHead = null, loTail = null;
                        // 高位链表：存放扩容之后的数组的下标位置，当前数组的下标位置 + 扩容之前的数组的长度（/image/hashmap/图来解释）
                        Node<K, V> hiHead = null, hiTail = null;
                        //下一个节点
                        Node<K, V> next;
                        do {
                            //开始遍历元素
                            next = e.next;
                            // oldCap 一定是1000...这样形式的（2的次幂，最高位一定是 1 ）
                            // 在下标为15的位置存在5个元素，而原来的数组的长度是 16 的二进制为10000；
                            // 上面的注释中有一句话是这么说的：e.hash 有两种情况，低位不用管，怎么都是0，高位可能是 1 也可能是 0，
                            // 如果是 1 那么结果就是 1，那该条件就不成立了，如果是 0 那么结果必然是 0。
                            // 在 jdk7 中 是需要重新计算hash位,
                            // 但是 jdk8 做了优化, 通过(e.hash & oldCap) == 0来判断是否需要移位;
                            // 如果为真则在原位不动, 否则则需要移动到当前hash槽位 + oldCap的位置；
                            if ((e.hash & oldCap) == 0) {
                                //如果改位置为空，直接将e放进去
                                if (loTail == null) {
                                    loHead = e;
                                } else {
                                    //否则就添加到链表的后面
                                    loTail.next = e;
                                }
                                loTail = e;
                            } else {
                                //到这一步说明高位1为1，添加也是如果原来位置没有元素那么就直接添加，
                                if (hiTail == null) {
                                    hiHead = e;
                                } else {
                                    //原来位置有值就将新元素添加到链表的尾部
                                    hiTail.next = e;
                                }
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        //首先此时原来的某个桶位已经链化了，这样子就可以推断出该桶位的所有的 Node 的 key 的二进制的低位都是相同的。
                        //假设我们桶的下标为15是链表，
                        // 而计算元素的下标就是根据 key 经过扰动（
                        // 扰动就是 h = key.hash ^ h >>> 16）hash 值与上桶的长度减一，
                        // 即 h & (table.length -1 )，
                        // 而现在桶的长度是 16 减一 就是15 ，
                        // 转成二进制就是 1111（这就是低四位），
                        // 高位全部补0即可，即 0 1111 ，
                        // 因为最终得到的下标全是相同的，在这种情况下Node中的key的hash计算出来的低位一定是相同的，
                        // 不然结果肯定不可能为一样的，但是Node中的key的hash高位不一定是相同的，
                        // 那为什么与上01111还能得到相同结果？
                        // 因为此时Node的高位不同（可能是0 也可能是1），
                        // 但是table-1的二进制数的高位是0，
                        // 所以此时是不受Node高位的hash值影响的，
                        // 所以在扩容以后，原来的如果高位是0的，那么在迁移到新的表中结果依旧是在同样的位置，
                        // 如果是高位是 1 ，那么迁移后的元素在桶中的位置就是 原来的桶长度 + 原来的元素的位置。

                        // 假设原来散列表的长度是16，length - 1转成二进制是 0 1111，
                        // 现在假设有一个 A 和 B 两个 Node 的 key 的 hash 值分别为：0 1111,1 1111
                        // A 和 0 1111 取余 结果是：0 1111 & 0 1111 = 0 1111，下标是15，
                        // B 1 1111 & 0 1111 = 0 1111，
                        // 这个时候是在原来的桶中的，
                        // 现在散列表扩容后长度变成了 32 ，
                        // 32 - 1 = 31 = 1 1111，
                        // 此时再来计算 A 和 B 的在新的散列表中的位置，
                        // A ：0 1111 & 1 1111 = 0 1111 = 15，
                        // 也就是说 A 在迁移到新的桶中的下标位置还是 15 ，再来看下
                        // B ：1 1111 & 1 1111 = 1 1111 = 31，即
                        // B 在新的散列表中的位置为
                        // 原来的散列表的长度（16）+ 原来的下标的位置（15） = 新的下标的位置（31），
                        // 这就是迁移后元素存放的特点。

                        //低位链表有数据
                        if (loTail != null) {
                            //将原来的低位链表的next置空，方便GC，
                            loTail.next = null;
                            //将低位链表直接添加到新的散列表的和原来的一样的下标位置
                            newTab[j] = loHead;
                        }
                        //高位链表有数据
                        if (hiTail != null) {
                            //将原来的高位链表的next置空，方便GC
                            hiTail.next = null;
                            //将高位链表的放在 新的散列表的 老表的长度+老表的位置 的下标位置
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }

```
扩容的注释写的非常清楚了，但是其中一个细节还是要详细讲一下
扩容过程中 是怎么处理链表的，
扩容时 HashMap的table扩大一倍，所以之前计算h&(length-1)时 生效的几位的前一位也要参与计算
0 1111和
1 1111
扩容前，寻址 位置假设都是1；
扩容后，高位的0和1参与运算 这两个hash的node就会放入不同的桶里
相差多少呢 相差1 0000 即oldCap大小  代码中为 `newTab[j + oldCap] = hiHead;`

## get()

```java
    public V get(Object key) {
        Node<K, V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
 /**
     * Implements Map.get and related methods
     *
     * @param hash hash for key
     * @param key  the key
     * @return the node, or null if none
     */
    final Node<K, V> getNode(int hash, Object key) {
        //当前HashMap的散列表的引用
        Node<K, V>[] tab;
        //first：桶头元素
        //e：用于存放临时元素
        Node<K, V> first, e;
        //n：table 数组的长度
        int n;
        //元素中的 k
        K k;
        // 将 table 赋值为 tab，不等于null 说明有数据，(n = tab.length) > 0 同理说明 table 中有数据
        //同时将 改位置的元素 赋值为 first
        if ((tab = table) != null && (n = tab.length) > 0 && (first = tab[(n - 1) & hash]) != null) {
            //定位到了桶的到的位置的元素就是想要获取的 key 对应的，直接返回该元素
            if (first.hash == hash && ((k = first.key) == key || (key != null && key.equals(k)))) {
                return first;
            }
            //到这一步说明定位到的元素不是想要的，且改位置不仅仅有一个元素，需要判断是链表还是树
            if ((e = first.next) != null) {
                //是否已经树化，本文不考虑
                if (first instanceof TreeNode) {
                    return ((TreeNode<K, V>) first).getTreeNode(hash, key);
                }
                //处理链表的情况
                do {
                    //如果遍历到了就直接返回该元素
                    if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k)))) {
                        return e;
                    }
                } while ((e = e.next) != null);
            }
        }
        //遍历不到返回null
        return null;
    }
```


## remove （）

```java
    public V remove(Object key) {
        Node<K, V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
                null : e.value;
    }
    
    @Override
    public boolean remove(Object key, Object value) {
        return removeNode(hash(key), key, value, true, true) != null;
    }
```

```java
	/**
     * Implements Map.remove and related methods
     *
     * @param hash       hash for key
     * @param key        the key
     * @param value      the value to match if matchValue, else ignored
     * @param matchValue if true only remove if value is equal
     * @param movable    if false do not move other nodes while removing
     * @return the node, or null if none
     */
    final Node<K, V> removeNode(int hash, Object key, Object value, boolean matchValue, boolean movable) {
        //当前HashMap 中的散列表的引用
        Node<K, V>[] tab;
        //p：表示当前的Node元素
        Node<K, V> p;
        // n：table 的长度
        // index：桶的下标位置
        int n, index;
        //(tab = table) != null && (n = tab.length) > 0 条件成立，说明table不为空（table 为空就没必要执行了）
        // p = tab[index = (n - 1) & hash]) != null 将定位到的捅位的元素赋值给 p ，并判断定位到的元素不为空
        if ((tab = table) != null && (n = tab.length) > 0 && (p = tab[index = (n - 1) & hash]) != null) {
            //进到 if 里面来了，说明已经定位到元素了
            //node：保存查找到的结果
            //e：表示当前元素的下一个元素
            Node<K, V> node = null, e;
            K k;
            V v;
            // 该条件如果成立，说明当前的元素就是要找的结果（这是最简单的情况，这个是很好理解的）
            if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k)))) {
                node = p;
            }
            //到这一步，如果 (e = p.next) != null 说明该捅位找到的元素可能是链表或者是树，需要继续判断
            else if ((e = p.next) != null) {
                //树，不考虑
                if (p instanceof TreeNode) {
                    node = ((TreeNode<K, V>) p).getTreeNode(hash, key);
                }
                //处理链表的情况
                else {
                    do {
                        //如果条件成立，说明已经匹配到了元素，直接将查找到的元素赋值给 node，并跳出循环（总体还是很好理解的）
                        if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        //将正在遍历的当前的临时元素 e 赋值给 p
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            // node != null 说明匹配到了元素
            // matchValue为false ，所以!matchValue  = true，后面的条件直接不用看了

            // 值匹配的时候 matchValue = true，所以 !matchValue = false,所以此时必须保证后面的值是true 才执行真正的 remove 操作
            if (node != null && (!matchValue || (v = node.value) == value || (value != null && value.equals(v)))) {
                //树，不考虑
                if (node instanceof TreeNode) {
                    ((TreeNode<K, V>) node).removeTreeNode(this, tab, movable);
                }
                // 这种情况是上面的最简单的情况
                else if (node == p) {
                    //直接将当前节点的下一个节点放在当前的桶位置（注意不是下一个桶位置，是该桶位置的下一个节点）
                    tab[index] = node.next;
                } else {
                    //说明定位到的元素不是该桶位置的头元素了，那直接进行一个简单的链表的操作即可
                    p.next = node.next;
                }
                //移除和添加都属于结构的修改，需要同步自增 modCount 的值
                ++modCount;
                //table 中的元素个数减 1
                --size;
                //啥也没做，不用管
                afterNodeRemoval(node);
                //返回被移除的节点元素
                return node;
            }
        }
        //没有匹配到返回null 即可
        return null;
    }
```
