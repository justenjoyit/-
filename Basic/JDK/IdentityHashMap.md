# IdentityHashMap

## 一、简介

1. 在比较key时，是比较 **引用-相等（reference-equality）** 而不是 **对象-相等（object-equality）**，也就是只有当key1==key2的时候才认为该键值相等。在如**HashMap**里，认为两个键值相等的方法为```k1 == null ? k2 == null : k1.equals(k2)```

2. 这个类的典型应用：

   2.1 拓扑保持对象图转换（如序列化、深拷贝）。为了完成转换，程序必须维护一个保存了所有已经执行过的对象引用的路径的节点表。

   2.2 代理对象

3. 允许**null** value和key
4. 拥有一个调整参数：**expected maximum size**
5. 非**synchronized**
6. **fail-fast**
7. 这是一个**线性探针（linear-probe）**哈希表
8. 对于很多JRE实现和混合操作，这个类比**HashMap**（使用链表而不是线性探针）有更好的性能

## 二、解读

1. 默认容量为32，默认期待最大容量为21，最小容量为4，最大容量为1<<29，但是实际上不能存多余1<<29-1的item，因为必须有一个槽用来存  **key == null**（为了避免get()、put()、remove()中的无限循环）

2. 用常量表示空键
	```java
	static final Object NULL_KEY = new Object();

    /**
     * Use NULL_KEY for key if it is null.
     */
    private static Object maskNull(Object key) {
        return (key == null ? NULL_KEY : key);
    }
   ```
   
3. hash方法
	```System.identityHashCode()```方法与```Object.hashCode()```默认返回的值是一样的，都是对象内存地址的值
	
	```java
   private static int hash(Object x, int length) {
        int h = System.identityHashCode(x);
        // Multiply by -127, and left-shift to use least bit as part of hash
        return ((h << 1) - (h << 8)) & (length - 1);
	 }
   ```
   
4. put方法
	key就在value前一个格子里
	
	```java
    public V put(K key, V value) {
     final Object k = maskNull(key);
   
        retryAfterResize: for (;;) {
            final Object[] tab = table;
            final int len = tab.length;
         int i = hash(k, len);
	
   		//线性探测
            for (Object item; (item = tab[i]) != null;
                 i = nextKeyIndex(i, len)) {
                if (item == k) {
                    @SuppressWarnings("unchecked")
                        V oldValue = (V) tab[i + 1];
                    tab[i + 1] = value;
                    return oldValue;
                }
         }
   
            final int s = size + 1;
            // Use optimized form of 3 * s.
            // Next capacity is len, 2 * current capacity.
            if (s + (s << 1) > len && resize(len))
             continue retryAfterResize;
   
            modCount++;
            tab[i] = k;
            tab[i + 1] = value;
            size = s;
            return null;
        }
    }
   ```

