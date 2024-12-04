# HashMap提问
### Q:请解释一下HashMap的参数loadFactor，它的作用是什么？
    loadFactor表示HashMap的拥挤程度，影响hash操作到同一个数组位置的概率。默认loadFactor等于0.75，
    当HashMap里面容纳的元素已经达到HashMap数组长度的75%时，表示HashMap太挤了，需要扩容，
    在HashMap的构造器中可以定制loadFactor。
### Q:HashMap 初始化传入int值，是怎么获取大于该int值的最小2的N次方的？
```java
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
 static final int MAXIMUM_CAPACITY = 1 << 30;
```
    解析下此方法，需要用到具体的一个值，就能看到规律
    以cap=10为例，那么n=9 -> 二进制只取低4位 1001
    n的二进制变为  1100
    n |= n>>>2  -> 1100 | 1100 >>> 2 -> 1100 | 0011 -> 1111
    之后的数字都不变即 n= 1111  -> n =15
    
    由此可见，即使cap数字再大都会，一步一步的最终转换成  1111....  -> n+1 则对应这2的某次方
    到了 n>>>16后如果是大于 MAXIMUM_CAPACITY就直接取值MAXIMUM_CAPACITY

### Q: HashMap中为什么table长度为2的N次幂？
    首先由上一个问题，tableSizeFor返回的是2的N次幂。在代码中传入一对KV时，会先进行 hash(K) 得到一个int值再table[i = (n - 1) & hash]计算得到该KV对应的table的位置
    假设hash = 1111 1111 1111 1111 1111 0000 1110 1010   n = 16  则n-1 = 15  -> 0000 0000 0000 0000 0000 0000 0000 1111
    15          0000 0000 0000 0000 0000 0000 0000 1111
    hash        1111 1111 1111 1111 1111 0000 1110 1010
    &           0000 0000 0000 0000 0000 0000 0000 1010  =  10
    
    可以看到这样最终结果定位到了table[10]。通过这种方式，取hash的低位与table.length-1 做&操作，效率很高
    
    同时当table扩容之后，length变为 n=32 后 n-1=31  -> 0000 0000 0000 0000 0000 0000 0001 1111 
    
    31          0000 0000 0000 0000 0000 0000 0001 1111
    hash        1111 1111 1111 1111 1111 0000 1110 1010
    &           0000 0000 0000 0000 0000 0000 0000 1010  =  10  还是在table[10]的位置
    
    假设hash2 = 1111 1111 1111 1111 1111 0000 1111 1010
    31          0000 0000 0000 0000 0000 0000 0001 1111
    hash2       1111 1111 1111 1111 1111 0000 1111 1010
    &           0000 0000 0000 0000 0000 0000 0001 1010  =  2^4 + 10  还是在table[2^4 + 10]的位置
    这里看到规律是: 当扩容过后，table中的Node要么还在old下标中，要么移动到  old下标 + oldSize(此处例子中表示为2^4)  中去

### Q: HashMap中为什么如何计算Key的hash值？为什么要怎么计算hash值？
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
    key的hash方法，可以看到取值是 h ^ (h>>>16) 。而不是key本身的hashCode，这是由于之后代码中
    tab[i = (n - 1) & hash]中使用到 length & hash值
    由于int值为32bit的 以数字 1111 1111 1111 1111 1111 0000 1110 1010 为例
    
     1.  h= hashCode() 1111 1111 1111 1111 1111 0000 1110 1010
    
     2. h              1111 1111 1111 1111 1111 0000 1110 1010
        h>>>16         0000 0000 0000 0000 1111 1111 1111 1111
        h ^ h>>>16     1111 1111 1111 1111 1111 1111 0001 0101
    
     3. (n-1) & hash  0000 0000 0000 0000 0000 0000 0000 1111   n为table的length，为2的次方，减一之后表示为二进制为
                      1111 1111 1111 1111 1111 1111 0001 0101
                      0000 0000 0000 0000 0000 0000 0000 0101   =  5
    
     如果在代码中不使用h ^ (h>>>16)  而是返回h本身，那么
                   0000 0000 0000 0000 0000 0000 0000 1111
     (n-1) & hash  1111 1111 1111 1111 1111 0000 1110 1010
                   0000 0000 0000 0000 0000 0000 0000 1010
    使用的是hash值中的低位数作为table的下标，但是在大量数据的情况下，低位数碰撞的概率很高
    右移16位正好为32bit的一半，自己的高半区和低半区做异或，是为了混合原始哈希码的高位和低位，
    来加大低位的随机性。而且混合后的低位掺杂了高位的部分特征，使高位的信息也被保留下来
### Q: 什么时候链表会转换为红黑树？
    putVal()方法中有段代码，在判断binCount >= 8-1之后进入treeifyBin()方法，但是方法体中会判断table
    是否 < 64进行resize操作。其实和网络上大多数说法不对，真正转换为红黑树是两个条件
    1.链表长度达到8
    2.table的长度大于64
    对应的Node才会转换为红黑树
```java
    TREEIFY_THRESHOLD = 8;
    for (int binCount = 0; ; ++binCount) {
        if ((e = p.next) == null) {
            p.next = newNode(hash, key, value, null);
            if (binCount >= TREEIFY_THRESHOLD - 1) // 判断代码
                treeifyBin(tab, hash);
            break;
        }
        if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k))))
            break;
        p = e;
    }
    final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        //转红黑树前还需判断一次table长度
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY) 
            resize();
        else if{
            //
        }
    }
```  
    