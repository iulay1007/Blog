### LruCache

LruCache是Android 3.1所提供的一个缓存类，用于实现内存缓存。

LruCache是个泛型类，主要算法原理是把最近使用的对象用强引用存储在 LinkedHashMap 中。每次访问一个值时，它都会被移动到队列的头部。当一个值被添加到一个满的缓存队列中时，该队列末尾的值将被清除，并有可能成为垃圾收集的对象。

即当缓存满时，把最近最少使用的对象从内存中移除



### LruCache中的LinkedHashMap

LruCache内部用LinkedHashMap保存值。HashMap和双向链表合二为一即是LinkedHashMap。由于LinkedHashMap是HashMap的子类，所以LinkedHashMap自然会拥有HashMap的所有特性。比如，LinkedHashMap的元素存取过程基本与HashMap基本类似，只是在细节实现上稍有不同。当然，这是由LinkedHashMap本身的特性所决定的，因为它额外维护了一个双向链表用于保持迭代顺序。此外，LinkedHashMap可以很好的支持LRU算法。(Least Recently Used，也就是最近最少使用算法)

HashMap是无序的，也就是说，迭代HashMap所得到的元素顺序并不是它们最初放置到HashMap的顺序。HashMap的这一缺点往往会造成诸多不便，因为在有些场景中，我们确需要用到一个可以保持插入顺序的Map。庆幸的是，JDK为我们解决了这个问题，它为HashMap提供了一个子类 —— LinkedHashMap。虽然LinkedHashMap增加了时间和空间上的开销，但是它通过维护一个额外的双向链表保证了迭代顺序。特别地，该迭代顺序可以是插入顺序，也可以是访问顺序。因此，根据链表中元素的顺序可以将LinkedHashMap分为：保持插入顺序的LinkedHashMap 和 保持访问顺序的LinkedHashMap，其中LinkedHashMap的默认实现是按插入顺序排序的。

**LruCache的构造函数**

```java
public LruCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
    }
```

其中就构造了一个LinkedHashMap

> LinkedHashMap<K,V>(int initialCapacity, float loadFactor, boolean accessOrder)

- `InitialCapacity`该参数设定实现散列集的数组链表中数组的长度，默认大小为16之后每超过指定的大小都扩展为2的n次方倍

- `loadFactor` 默认为0.75f，注意是float类型的

- 当`accessOrder`为**true**的时候，在我们访问了一个Entry<K,V>时，我们会调用`afterNodeAccess()`方法，将我们当前访问的节点放入到链表的末尾，利用这个特性便可以区分谁是最近访问，谁是最近最不常访问元素了



  boolean removeEldestEntry(Map.Entry)该方法返回值为true时，会删除最近最不常使用的元素，也就是double-link的头部节点，当插入一个新节点之后`removeEldestEntry()`方法会被`put()、putAll()`方法调用，我们可以通过override该方法，来控制删除最旧节点的条件



  **LruCache中的maxSize，在没有重写sizeOf方法的时候，这是缓存项的最大数目和。**



  以用户定义的单位返回条目的大小。默认实现返回1，因此size表项数，maxsize为最大表项数。

  当条目在缓存中时，它的大小不能改变。

  ```java
  protected int sizeOf(K key, V value) {
          return 1;
      }
  ```

  > 此缓存的size。不一定是元素的数量。默认情况下，缓存大小以条目数度量。覆盖
  >
  > sizeOf()方法以不同的单位调整缓存的大小

  例如，这个缓存限制为4MiB的bitmaps:：

  ```java
  {
      int cacheSize = 4 * 1024 * 1024; // 4MiB
      LruCache<String, Bitmap> bitmapCache = new LruCache<String, Bitmap>(cacheSize) {
          protected int sizeOf(String key, Bitmap value) {
              return value.getByteCount();//返回字节数,也可根据需要自行重写
          }
      }}
  ```



**resize方法**

即重新定义maxSize

```java
  public void resize(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }

        synchronized (this) {
            this.maxSize = maxSize;
        }
        trimToSize(maxSize);
    }
```

**trimToSize**方法

```java
 public void trimToSize(int maxSize) {
     //循环删除最老的条目直到size小于maxSize
        while (true) {
            K key;
            V value;
            synchronized (this) {
                if (size < 0 || (map.isEmpty() && size != 0)) {
                    throw new IllegalStateException(getClass().getName()
                            + ".sizeOf() is reporting inconsistent results!");
                }

                if (size <= maxSize) {
                    break;
                }

                Map.Entry<K, V> toEvict = map.eldest();
                if (toEvict == null) {
                    break;
                }

                key = toEvict.getKey();
                value = toEvict.getValue();
                map.remove(key);
                size -= safeSizeOf(key, value);
                evictionCount++;
            }

            entryRemoved(true, key, value, null);
        }
    }

```

> Map.Entry是Map声明的一个内部接口，此接口为泛型，定义为Entry<K,V>。它表示Map中的一个实体（一个key-value对）。接口中有getKey(),getValue等方法



safeSizeOf本质还是调用sizeOf

```java
  private int safeSizeOf(K key, V value) {
        int result = sizeOf(key, value);
        if (result < 0) {
            throw new IllegalStateException("Negative size: " + key + "=" + value);
        }
        return result;
    }
```



> protected void entryRemoved(boolean evicted, K key, V oldValue, V newValue) {}

这个方法是当一个值被回收以腾出空间（如trimToSize方法）或调用put、remove时被调用。如果缓存值中包含需要显式释放的资源，可以重写entryRemoved（）方法，在其中完成资源的回收工作。

参数含义：

*evicted*：如果要删除条目以腾出空间，则为true；如果删除是由put（）、get（）或remove（）引起的，则为false。

*newValue*：如果它存在的话且非null，则由put或get导致删除。如果为null，它是由回收或remove引起的。



**get方法**

返回key对应的值（如果它存在于缓存中或可以由create（）方法创建）。如果返回了一个值，它将移动到队列的头部。如果值未缓存且无法创建，则返回null

```java
 public final V get(K key) {
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        V mapValue;
        synchronized (this) {
            mapValue = map.get(key);
            if (mapValue != null) {
                hitCount++;
                return mapValue;
            }
            missCount++;
        }

        /*
         * Attempt to create a value. This may take a long time, and the map
         * may be different when create() returns. If a conflicting value was
         * added to the map while create() was working, we leave that value in
         * the map and release the created value.
         */
		/*试图创造值。这可能需要很长时间，并且当create（）返回时map可能不同。如果在create（）工		作时向映射添加了一个冲突的值，则将该值保留在map并释放所创建的值。*/
     
        V createdValue = create(key);
        if (createdValue == null) {
            return null;
        }

        synchronized (this) {
            createCount++;
            mapValue = map.put(key, createdValue);

            if (mapValue != null) {
                // There was a conflict so undo that last put
                map.put(key, mapValue);
            } else {
                size += safeSizeOf(key, createdValue);
            }
        }

        if (mapValue != null) {
            entryRemoved(false, key, createdValue, mapValue);
            return mapValue;
        } else {
            trimToSize(maxSize);
            return createdValue;
        }
    }

```

**create**方法

如果无法根据某个key得到值，可以用create方法根据key得到指定值，create方法默认返回null，可以自行override

```java
  protected V create(K key) {
        return null;
    }
```

### **end**

LruCache里面其实没有很复杂的代码，它复杂的地方还是在于用LinkedHashMap实现LRU算法。