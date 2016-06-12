
**面试题： Java中ArrayList和LinkedList的主要区别是什么？**

这个问题首先要知道数组和链表的特点
数组的特点：寻址容易，插入和删除困难。
链表的特点是：寻址困难，插入和删除容易。
ArrayList的底层实现就是通过动态数组来实现的，LinkedLIst底层实现就是通过链表来实现的，所以直接答出数组和链表的特点就ok

**面试题： hashMap是怎样实现key-value这样键值对的保存？**

HashMap中有一个内部类Entry，

```java
 static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        Entry<K,V> next;
        int hash;
        //.....
}
```

主要有4个属性，key ,hash,value,指向下一个节点的引用next ，看到这个实体类就明白了，在HashMap中存放的key-value实质是通过实体类Entry来保存的

**面试题： hashMap的实现原理？**

HashMap使用到的数据类型主要就是数组和链表，首先看原理图

![](http://img.blog.csdn.net/20140214101204468?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYXNkZXdxMzgwMzAzMzE4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

在hashMap的原理图中，左边是通过数组来存放链表的第一个节点，看懂这个图这个问题就ok

**面试题： hashMap的put过程？**

面我们提到过Entry类里面有一个next属性，作用是指向下一个Entry。比如说： 第一个键值对A进来，通过计算其key的hash得到的index=0，记做:Entry[0] = A。一会后又进来一个键值对B，通过计算其index也等于0，现在怎么办？HashMap会这样做:B.next = A,Entry[0] = B,如果又进来C,index也等于0,那么C.next = B,Entry[0] = C；这样我们发现index=0的地方其实存取了A,B,C三个键值对,他们通过next这个属性链接在一起。也就是说数组中存储的是最后插入的元素。

```java
     public V put(K key, V value) {
        if (key == null)
            return putForNullKey(value);
        int hash = hash(key);
        int i = indexFor(hash, table.length);
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {<span style="color:#ff0000;">//循环判断插入的key是否已经存在，若存在就更新key对应的value</span>
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }

        modCount++;
        addEntry(hash, key, value, i)<span style="color:#ff0000;">;//key不存在，那么插入新的key-value</span>
        return null;
    }
```

**面试题： hashMap的get过程？**

```java
public V get(Object key) {
        if (key == null)
            return getForNullKey();
        int hash = hash(key.hashCode());
        //先定位到数组元素，再遍历该元素处的链表
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k)))
                return e.value;
        }
        return null;
}
```

**面试题： hashMap存取的时候是如何定位数组下标的？**

找到indexFor这个方法，hashMap就是通过这个方法获取数组下标的：

```java
static int indexFor(int h, int length) {
        return h & (length-1);
    }
```

通过key的hash值和数组长度求&，这意味着数组下标相同，并不表示hashCode相同。所以在Entry中存有一个hash值，在比较Entry的时候都是想比较hash值

**面试题： hashMap中数组的初始化大小过程？**

```java
 public HashMap(int initialCapacity, float loadFactor) {
        .....// Find a power of 2 >= initialCapacity
        int capacity = 1;
        while (capacity < initialCapacity)
            capacity <<= 1;  //相当于capacity = capacity * 2
        this.loadFactor = loadFactor;
        threshold = (int)(capacity * loadFactor);
        table = new Entry[capacity];
        init();
    }
```

从上面的代码我们可以看出**数组的初始大小并不是构造函数中的initialCapacity！！而是2的n次方**

**面试题： hashMap什么时候开始rehash?**

在hashMap中有一个加载因子loadFactor，默认值是0.75，当数组的实际存入值的大小 > 数组的长度×loadFactor 时候就会rehash，重新创建一个新的表，将原表的映射到新表中，这个过程很费时。



