# ArrayList提问
### Q: 为什么ArrayList中存在MAX_ARRAY_SIZE=Integer.MAX_VALUE - 8
    数组在java里是一种特殊类型，java里数组不是类，所以也就没有对应的class文件，数组类型是由jvm从元素类型合成出来的；
    在jvm中获取数组的长度是用arraylength这个专门的字节码指令的；
    在数组的对象头里有一个_length字段，记录数组长度，只需要去读_length字段就可以了所以ArrayList中定义的最大长度为Integer最大值减8，
    这个8就是就是存了数组_length字段

### Q:为什么EMPTY_ELEMENTDATA和DEFAULTCAPACITY_EMPTY_ELEMENTDATA都是{}怎么不用同一个呢？
    可以查看到ArrayList的构造器，无参构造使用的是DEFAULTCAPACITY_EMPTY_ELEMENTDATA，
```java
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

```
    而有参构造里面传入initialCapacity=0的时候就会使用到EMPTY_ELEMENTDATA
```java
   public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                    initialCapacity);
        }
    }
```
    在calculateCapacity()中可以看到只针对了DEFAULTCAPACITY_EMPTY_ELEMENTDATA判断
    如果使用initialCapacity=0构造函数，添加一个元素后，elementData.length=1
    如果使用无参构造，添加一个元素后，elementData.length=10 (DEFAULT_CAPACITY就是10）

### Q：为什么 elementData 使用 transient 修饰，该关键字声明数组默认不会被序列化？  
    ArrayList提供了两个用于序列化和反序列化的方法，readObject和writeObject,

### Q：为什么不直接用elementData来序列化，而采用上面的方式来实现序列化呢？
    elementData是一个缓存数组，默认size为10,对ArrayList进行add操作当空间不足时，会对ArrayList进行扩容。通常扩容的倍数为1.5倍。
    readObject和writeObject的方式来实现序列化时，就可以保证只序列化实际存储的那些元素，而不是整个数组，从而节省空间和时间