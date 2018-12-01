---
title: ArrayList原理及使用说明
date: 2018-11-28 16:13:36
tags:
categories:
- 数据结构
---
#### 写在前面
作为Android开发者，Java集合可能是开发中最常使用的类之一了。但很多人可能跟我一样，对Java集合只停留在“使用”的层面上，而对其的实现、原理如何只是略知一二，所以有时可能忽略了一些小细节。这些细节可能对项目的整体性能影响不大，但我觉得，要成为一个好的程序员，必须要精益求精，对代码性能“锱铢必较”。

举个例子，各位在创建ArrayList实例时有没有想过到底要不要指定其初始容量？指定了会怎样？不指定又会怎样？如果你跟博主我有同样的困惑，那么本文一定能给你个满意的答案！

#### 正文
ArrayLIst的细节

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/java/list.png)

细节1：ArrayList基于数组实现，访问元素效率快，插入删除元素效率慢
ArrayList是基于数组实现的，这个似乎不是什么秘密了，但为了文章的完整性，还是要介绍下。ArrayList内部维护一个数组elementData，用于保存列表元素，基于数组的数组这数据结构，我们知道，其索引元素是非常快的

```
public E get(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

    return (E) elementData[index]; // 索引无需遍历，效率非常高！
}

public E set(int index, E element) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

    E oldValue = (E) elementData[index];
    elementData[index] = element; // 索引无需遍历，效率非常高！
    return oldValue;
}
```

可以看到，get、set直接根据索引获取了目标元素，中间不用做任何的遍历操作，效率是非常快的。但是对于插入和删除操作效率就不太理想了：

```
public void add(int index, E element) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

    ensureCapacityInternal(size + 1);  // 先判断是否需要扩容
    System.arraycopy(elementData, index, elementData, index + 1, // 把index后面的元素都向后偏移一位
            size - index);
    elementData[index] = element;
    size++;
}
```

从插入操作的源码可以看到，插入前，要先判断是否需要扩容（扩容后面会讲，这里先跳过），然后把Index后面的元素都偏移一位，这里的偏移是需要把元素复制后，再赋值当前元素的后一索引的位置。显然，这样一来，插入一个元素，牵连到多个元素，效率自然就低了。再来看看删除操作：

```
public E remove(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

    modCount++;
    E oldValue = (E) elementData[index];

    int numMoved = size - index - 1;
    if (numMoved > 0) {
        // 把index后面的元素向前偏移一位，填补删除的元素
        System.arraycopy(elementData, index + 1, elementData, index,
                numMoved);
    }
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```

同样，删除一个元素，需要把index后面的元素向前偏移一位，填补删除的元素，也是牵连了多个元素。所以大家在使用时要谨慎了！
#### 细节2：ArrayList支持快速随机访问
什么是随机访问？我们不防先来看看ArrayList的类定义：

```
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```
看到RandomAccess了吗，这个就是支持快速随机访问的标记，我们再点进去看看其源码：

```
/**
 * ...
 * <p>It is recognized that the distinction between random and sequential
 * access is often fuzzy.  For example, some <tt>List</tt> implementations
 * provide asymptotically linear access times if they get huge, but constant
 * access times in practice.  Such a <tt>List</tt> implementation
 * should generally implement this interface.  As a rule of thumb, a
 * <tt>List</tt> implementation should implement this interface if,
 * for typical instances of the class, this loop:
 * <pre>
 *     for (int i=0, n=list.size(); i &amp;lt; n; i++)
 *         list.get(i);
 * </pre>
 * runs faster than this loop:
 * <pre>
 *     for (Iterator i=list.iterator(); i.hasNext(); )
 *         i.next();
 * </pre>
 * ...
 */
public interface RandomAccess {
}
```

额，是一个接口，没有任何的属性或方法定义。其实它只是一个标记，继承于它就相当于告诉别人，我支持快速随机访问，上面代码我特意留下部分的注释说明，其中关键的部分在说，**通常情况下，使用索引访问的效率比使用迭代器访问的效率快！**

我们把目光暂时转移到Collections类下，其中有很多基于是否有继承于RandomAccess的List做不同的算法选择判断，我们来看其中的二分查找算法：

```
public static <T> int binarySearch(List<? extends Comparable<? super T>> list, T key) {
    if (list instanceof RandomAccess || list.size()<BINARYSEARCH_THRESHOLD)
        // 当List实现了RandomAccess或小于一定阀值时，使用索引二分查找算法
        return Collections.indexedBinarySearch(list, key);
    else
        return Collections.iteratorBinarySearch(list, key);
```

所以快速随机访问是针对于Collections中的方法而言的（其他类是否也有？欢迎大神们补充），支持快速随机访问时，就选择索引访问，效率会很快。

另外，从上面的二分查找算法我们又能得到一个提高效率的小细节：我们知道List是提供了IndexOf和lastIndexOf方法来检索元素的，它们分别是从头和尾开始，一个一个比较的，那么显然，使用Collections#binarySearch在大多数情况效率会比
IndexOf和lastIndexOf更快～

#### 细节3：大多数情况下，我们都应该指定ArrayList的初始容量
如果说上面所介绍的细节大部分童鞋都知道，那这个细节相信很多人都不知道，包括在看源码之前的我。在讲为什么之前，我们需要先来了解ArrayList的扩容机制。

ArrayList每次扩容至少为原来容量大小的1.5倍，其默认容量是10，当你不为其指定初始容量时，它就会创建默认容量大小为10的数组：

```
// 默认最小容量
private static final int DEFAULT_CAPACITY = 10;

// 空数组
private static final Object[] EMPTY_ELEMENTDATA = {};

// 默认容量空数组，可以理解为一个标记
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

// 指定最小容量创建列表
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

// 创建默认空列表
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA; // 默认容量空数组
}
```

我们经常使用ArrayList的默认构造函数来创建实例，等等，不是说不指定初始容量会创建默认容量大小为10的数组吗？但这里只赋值了空数组。是的，还记得我们上面分析的add源码有个扩容操作吗？**如果使用默认构造函数来创建实例，在第一次添加元素时，就会进行扩容，扩容到默认容量10的数组：**

```
// 每次添加元素都会调用
private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        // 如果为默认容量空数组的话，添加元素时，至少扩容到默认最小容量
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }

    ensureExplicitCapacity(minCapacity);
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0) // 大于当前容量就扩容
        grow(minCapacity);
}

// 扩容
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1); // 1.5倍原来大小
    // 先尝试扩容到1.5倍原来容量的大小，如果比用户指定的大，那么就扩容1.5倍
    // 否则扩容用户指定的
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

所谓“扩容”就是创建一个长度更大的数组，再把旧数组的元素全部赋值到新数组。显然，这个操作效率也是不理想的。虽然使用默认构造函数创建的实例，在第一次添加元素的扩容并没有元素复制，但还是要另外创建一个数组，并且是大小为10的数组，可能你并不需要这么大的数组，可能是3，可能是5，那么我们为何不一开始就指定其容量呢？

指定初始容量的方法也很简单，我们使用带int参数的构造函数就可以了：

```
// 指定最小容量创建列表
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

或者有童鞋会说，使用ensureCapacity指定容量也行，其实不然，为何ensureCapacity对容量大小有限制：

```
// 指定最小容量
public void ensureCapacity(int minCapacity) {
    int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
            // any size if not default element table
            ? 0
            // larger than default for default empty table. It's already
            // supposed to be at default size.
            : DEFAULT_CAPACITY;

    // 指定最小容量成功的情况
    // 1.使用 new ArrayList() 创建实例并添加元素前，指定容量大小不能小于默认容量10
    // 2.列表已存在元素，指定容量大小不能小于当前容量大小
    if (minCapacity > minExpand) {
        ensureExplicitCapacity(minCapacity);
    }
}
```

所以讲到这，相信大家有答案了，**为什么创建ArrayList要指定其初始容量？显然我们是不希望它进行耗时的扩容操作，并且能在我们预知的情况下尽量使用大小刚刚好的列表，而不浪费任何资源。**那么我们可以得到以下经验：

都不应该使用默认构造函数创建实例，以免自动扩容到默认最小容量（10）

当列表容量确定，应该指定容量的方式创建实例

当列表容量不确定时，可以预估我们将有会多少元素，指定稍大于预估值的容量

#### LinkedList的细节
再来看看我们熟悉的LinkedList的细节～
细节1：LinkedList基于链表实现，插入删除元素效率快，访问元素效率慢
LinkedList内部维护一个双端链表，可以从头开始检索，也可以从尾开始检索。同样的，得益于链表这一数据结构，LinkedList在插入和删除元素效率非常快。

插入元素只需新建一个node，再把前后指针指向对应的前后元素即可：

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/java/list111.png)

```
// 链尾追加
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}

// 指定节点前插入
void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    // 插入节点，succ为Index的节点，可以看到，是插入到index节点的前一个节点
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}

public void add(int index, E element) {
    checkPositionIndex(index);

    if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index));
}
```

同样，删除元素只要把删除节点的链剪掉，再把前后节点连起来就搞定了：

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/java/list2323.png)

```
E unlink(Node<E> x) {
    // assert x != null;
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;

    if (prev == null) {
        // 链头
        first = next;
    } else {
        prev.next = next;
        x.prev = null;
    }

    if (next == null) {
        // 链尾
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }

    x.item = null;
    size--;
    modCount++;
    return element;
}

public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}
```

但由于链表我们只知道头和尾，中间的元素要遍历获取的，所以导致了访问元素时，效率就不好了：

```
Node<E> node(int index) {
    // 使用了二分法
    if (index < (size >> 1)) { // 如果索引小于二分之一，从first开始遍历
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else { // 如果索引大于二分之一，从last开始遍历
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}

public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}
```

所以，LinkedList和ArrayList刚好是互补的，所以具体场景，应考虑哪种操作最频繁，从而选择不同的List来使用。

#### 细节2：LinkedList可以当作队列和栈来使用
不知大家有没注意到在图2.2中，LinkedList非常“特立独行地”继承了Deque接口，而Deque又继承于Queue接口，这队列和栈的方法定义就是在这些接口中定义的，而LinkedList实现其方法，使自身具备了队列的栈的功能。
当作队列（先进先出）使用：

```
// 进队
public boolean offerFirst(E e) {
    addFirst(e);
    return true;
}

// 出队
public E pollLast() {
    final Node<E> l = last;
    return (l == null) ? null : unlinkLast(l);
}
```

当作栈（后进又出）来使用：

```
// 进栈
public void push(E e) {
    addFirst(e);
}

// 出栈，如果为空列表，会抛出异常
public E pop() {
    return removeFirst();
}
```

#### SynchronizedList的细节
在Collections类中提供了很多线程线程的集合类，其实他们实现很简单，只是在集合操作前，加一个锁而已。

细节1：SynchronizedList继承于SynchronizedCollection，使用装饰者模式，为原来的List加上锁，从而使List同步安全
先来看下SynchronizedCollection的定义：

```
static class SynchronizedCollection<E> implements Collection<E>, Serializable {
    private static final long serialVersionUID = 3053995032091335093L;

    final Collection<E> c; // 装饰的集合
    final Object mutex; // 锁

    SynchronizedCollection(Collection<E> c) {
        this.c = Objects.requireNonNull(c);
        mutex = this;
    }

    SynchronizedCollection(Collection<E> c, Object mutex) {
        this.c = Objects.requireNonNull(c);
        this.mutex = Objects.requireNonNull(mutex);
    }
}
```

可以看到，可以指定一个对象作为锁，如果不指定，默认就锁了集合了。
再来看下我们关注的SynchronizedList：

```
static class SynchronizedList<E>
    extends SynchronizedCollection<E>
    implements List<E> {

    final List<E> list;

    SynchronizedList(List<E> list) {
        super(list);
        this.list = list;
    }
    SynchronizedList(List<E> list, Object mutex) {
        super(list, mutex);
        this.list = list;
    }

    ...

    public E get(int index) {
        synchronized (mutex) {return list.get(index);}
    }
    public E set(int index, E element) {
        synchronized (mutex) {return list.set(index, element);}
    }
    public void add(int index, E element) {
        synchronized (mutex) {list.add(index, element);}
    }
    public E remove(int index) {
        synchronized (mutex) {return list.remove(index);}
    }

    ...
}
```

#### 写在最后
关于我们经常使用的List的细节到此就介绍完了，如果上面我有言论有误或不严谨的，欢迎大家指正；如果有另外一些细节我没谈及到的，也欢迎大神们补充。

最后，我们来做一次总结：

**ArrayList和LinkedList适用于不同使用场景，应根据具体场景从优选择**

**根据ArrayList的扩容机制，我们应该开始就指定其初始容量，避免资源浪费**

**LinkedList可以当作队列和栈使用，当然我们也可以进一步封装**

**尽量不使用Vector和Stack，同步场景下，使用SynchronizedList替代**
