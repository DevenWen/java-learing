# CopyOnWriteArrayList

## 前言
我是想知道，为什么在`foreach`循环中，`CopyOnWriteArrayList` 可以增减对象。

`CopyOnWriteArrayList ` 是一个适合多读少写场景的 `List`。它在读数据是完全不设锁的。在一般的列表中，为了保证读取正确，假如遇上写操作，则读操作也会被阻塞。但是 `COWArrayList` 阻塞相关读操作。在写的瞬间会拷贝一个数组副本，在`volatile`语义下安全的替换掉旧值。


## 分析
### 0 接口分析
```java
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

`RandomAccess` 是 `List` 实现所使用的标记接口，用来表明其支持快速（通常是固定时间）随机访问。此接口的主要目的是允许一般的算法更改其行为，从而在将其应用到随机或连续访问列表时能提供良好的性能。

所以它只是一个标记的作用。


### 1 类的字段分析
```java
/** 
 *	The lock protecting all mutators 
 * 用于对所有的改变操作进行锁处理
 */
final transient ReentrantLock lock = new ReentrantLock();

/** The array, accessed only via getArray/setArray. */
private transient volatile Object[] array;
```

### 2 构造
```java
/**
 *
 * @param c the collection of initially held elements
 * @throws NullPointerException if the specified collection is null
 */
public CopyOnWriteArrayList(Collection<? extends E> c) {
    Object[] elements;
    if (c.getClass() == CopyOnWriteArrayList.class)
        elements = ((CopyOnWriteArrayList<?>)c).getArray();
    else {
        elements = c.toArray();
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elements.getClass() != Object[].class)
            elements = Arrays.copyOf(elements, elements.length, Object[].class);
    }
    setArray(elements);
}
```
这个构造方法中，特意区分了 COWList 和其他 Collection 的处理方式，
假如 Collection 是一个 COWlist ，那么直接复制一个副本过来即可。因为在修改后，这个数据也会改变的，并不会原来的array有任何的改变，安全同时由最快速的方法。

`Arrays.copyOf` 是常用的用于复制的数组的方法。内部调用的时 System 本地方法。以达到速度最大化。

```java
public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) 
```

> 复制指定的数组，使用空值截断或填充（如有必要），以使副本具有指定的长度。对于在原始数组和副本中都有效的所有索引，这两个数组将包含相同的值。对于在副本中有效但不在原始副本中的任何索引，副本将包含null。当且仅当指定的长度大于原始数组的长度时，这些索引才会存在。结果数组是newType类。（来自谷歌翻译 jdk8 ）


### 3 clone
```java
/**
 * Returns a shallow copy of this list.  (The elements themselves
 * are not copied.)
 *
 * @return a clone of this list
 */
public Object clone() {
    try {
        @SuppressWarnings("unchecked")
        CopyOnWriteArrayList<E> clone =
            (CopyOnWriteArrayList<E>) super.clone();
        clone.resetLock();
        return clone;
    } catch (CloneNotSupportedException e) {
        // this shouldn't happen, since we are Cloneable
        throw new InternalError();
    }
}
```
它的拷贝方法并没有进行深度拷贝。但在拷贝锁对象的时候，对副本使用了 `resetLock`方法。

```Java
// Support for resetting lock while deserializing
    private void resetLock() {
        // 通过volatile可见的方式，线程安全地发布一个新的锁
        // 同时这也是一个cas原子操作
        // jdk 设置obj对象中offset偏移地址对应的整型field的值为指定值。支持volatile store语义
        UNSAFE.putObjectVolatile(this, lockOffset, new ReentrantLock());
    }
    private static final sun.misc.Unsafe UNSAFE;
    private static final long lockOffset;  // 锁的偏移量
    static {
        try {
        // 静态代码块会计算锁的偏移值
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> k = CopyOnWriteArrayList.class;
            lockOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("lock"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
```

## 4 Set 方法
```java

/**
 * Replaces the element at the specified position in this list with the
 * specified element.
 *
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public E set(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        E oldValue = get(elements, index);

        if (oldValue != element) {
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len);
            newElements[index] = element;
            setArray(newElements);
        } else {
            // Not quite a no-op; ensures volatile write semantics 
            setArray(elements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}

```
`Not quite a no-op; ensures volatile write semantics` 在上述的方法里，对于 `oldValue == element` 的值，也就是新值和旧值是一样时，代码并没有不做工作，而是再一次写入。`setArray` 的主要目的是建立 `happen-before` 关系。

参考：
[Why setArray() method call required in CopyOnWriteArrayList](https://stackoverflow.com/questions/28772539/why-setarray-method-call-required-in-copyonwritearraylist)


## 5 remove 
```java
/**
 * A version of remove(Object) using the strong hint that given
 * recent snapshot contains o at the given index.
 */
private boolean remove(Object o, Object[] snapshot, int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] current = getArray();
        int len = current.length;
        if (snapshot != current) findIndex: {
            int prefix = Math.min(index, len);
            for (int i = 0; i < prefix; i++) {
                if (current[i] != snapshot[i] && eq(o, current[i])) {
                    index = i;
                    break findIndex;
                }
            }
            if (index >= len)
                return false;
            if (current[index] == o)
                break findIndex;
            index = indexOf(o, current, index, len);
            if (index < 0)
                return false;
        }
        Object[] newElements = new Object[len - 1];
        System.arraycopy(current, 0, newElements, 0, index);
        System.arraycopy(current, index + 1,
                         newElements, index,
                         len - index - 1);
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}

```
删除的操作，在上锁后，先对比前面那个的数据`snapshot`是否被改变过，假如被改变了，就需要在重新在`current`数组中寻找下标，假如寻找不到，就视为删除失败。

## 6 其余的批量删除和添加也大致如此
## 7 COWSubList
```java
private static class COWSubList<E>
        extends AbstractList<E>
        implements RandomAccess {
        	// code...
        }
```
在`list`中，读取子列表后，对于子列表的操作是会影响旧的操作。这个特性和 COW 中会显得更加艰难。COW 本身就是通过不断拷贝副本来保证数据一致性的。它的子列表要兼容这种特性，意味着它要持有原列表的对象和锁，对于子列表的修改，是会映射到原列表那去的。

## 8 回答前言的问题
> 为什么`CopyOnWriteArrayList `可以一边遍历，一边删改？

首先是因为`static final class COWIterator<E>`的实现，列表在产生迭代器后，后面的操作都是基于副本去进行的。所以不报 `ConcurrentModificationException` 异常。

其实这个异常也是工程师对列表安全性和正确性保障而设定的，而不是系统自发的异常。在清除对象被修改后，使程序失败是更好地处理方法。
