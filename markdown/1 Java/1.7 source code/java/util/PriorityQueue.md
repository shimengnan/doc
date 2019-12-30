# PriorityQueue

## 摘要

最小堆,add siftUp, take siftDown,remove 先siftDown 在siftUp

## 源码

```java
package java.util;

import java.util.function.Consumer;

/**
 * maximumsize Integer.MAX_VALUE-8 JVM在创建数组时会预留header words,过大会OutOfMemoryError
 * 不是线程安全的.多线程操作下使用PriorityBlockingQueue
 * 当前实现  add,offer,poll,remove 时间复杂度为O(log(n)) 
 *		    remove,constains	 时间复杂度 liner time 线性时间
 *   		peek,element,size 	 常量时间
 * @since 1.5
 * @author Josh Bloch, Doug Lea
 * @param <E> the type of elements held in this collection
 */
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable {

    private static final long serialVersionUID = -7720805057305804111L;

    private static final int DEFAULT_INITIAL_CAPACITY = 11;

    /**
     * 优先级队列 是一个 平衡二叉堆.
     * 用数组表示时, 当前节点下标为n,左子节点为 2*n+1 右子节点 (2+1)*n
     */
    transient Object[] queue; // non-private to simplify nested class access

    /**
     * 优先级队列中元素的数量.
     */
    private int size = 0;

    /**
     * 排序规则,如果为空是用 元素的自然排序
     */
    private final Comparator<? super E> comparator;

    /**
     * 优先级队列的改动次数 用于抛出 ConcurrentModificationException
     */
    transient int modCount = 0; // non-private to simplify nested class access

    /**
     * 略
     */
    public PriorityQueue() {
        this(DEFAULT_INITIAL_CAPACITY, null);
    }

    /**
     * 略
     * @since 1.8
     */
    public PriorityQueue(Comparator<? super E> comparator) {
        this(DEFAULT_INITIAL_CAPACITY, comparator);
    }

    /**
     * 略
     */
    public PriorityQueue(int initialCapacity,
                         Comparator<? super E> comparator) {
        if (initialCapacity < 1)
            throw new IllegalArgumentException();
        this.queue = new Object[initialCapacity];
        this.comparator = comparator;
    }

    /**  其余构造方法 略*/
    

   

    /**
     * 数组的最大分配数量.一些MV 会预留 header words 给数组.
     * 尝试分配数量过大的数组 会引起 OutOfMemoryError
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    /**
     * 扩容
     * @param minCapacity the desired minimum capacity
     */
    private void grow(int minCapacity) {
        int oldCapacity = queue.length;
        // 64 一类 double,以上 每次增长 50% 
        int newCapacity = oldCapacity + ((oldCapacity < 64) ?
                                         (oldCapacity + 2) :
                                         (oldCapacity >> 1));
        // overflow-conscious code  显示溢出检查
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        queue = Arrays.copyOf(queue, newCapacity);
    }
	/**最大容量确认*/
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }

    /**
     * 插入元素到优先级队列
     * @throws ClassCastException,NullPointerException 
     */
    public boolean add(E e) {
        return offer(e);
    }

    /**
     * 插入元素到优先级队列
     * @throws ClassCastException,NullPointerException 
     */
    public boolean offer(E e) {
        if (e == null)
            throw new NullPointerException();
        modCount++;//操作数量+1
        int i = size;
        if (i >= queue.length)
        	//如果当前队列满,扩容
            grow(i + 1);
        //待添加元素下标
        size = i + 1;
        if (i == 0)
        	//数组为空,直接添加元素
            queue[0] = e;
        else
        	//将元素添加到位置i 同时向上调整
            siftUp(i, e);
        return true;
    }
	/** 返回优先级队列第一个元素 */
    @SuppressWarnings("unchecked")
    public E peek() {
        return (size == 0) ? null : (E) queue[0];
    }
	/** 返回元素下标 */
    private int indexOf(Object o) {
        if (o != null) {
            for (int i = 0; i < size; i++)
                if (o.equals(queue[i]))
                    return i;
        }
        return -1;
    }

    /**
     * 从队列中移除对象(通过.equals 来判定是否相同)
     */
    public boolean remove(Object o) {
        int i = indexOf(o);
        if (i == -1)
            return false;
        else {
            removeAt(i);
            return true;
        }
    }

    /**
     * 从队列中移除对象(使用直接应用移除)
     */
    boolean removeEq(Object o) {
        for (int i = 0; i < size; i++) {
            if (o == queue[i]) {
                removeAt(i);
                return true;
            }
        }
        return false;
    }

    /**
     * 判断队列中是否包含当前元素
     */
    public boolean contains(Object o) {
        return indexOf(o) != -1;
    }

    /**
     *返回队列数组元素,(非顺序) 
     * @return an array containing all of the elements in this queue
     */
    public Object[] toArray() {
        return Arrays.copyOf(queue, size);
    }

    /**
     * 略
     */
    @SuppressWarnings("unchecked")
    public <T> T[] toArray(T[] a) {
        final int size = this.size;
        if (a.length < size)
            // Make a new array of a's runtime type, but my contents:
            return (T[]) Arrays.copyOf(queue, size, a.getClass());
        System.arraycopy(queue, 0, a, 0, size);
        if (a.length > size)
            a[size] = null;
        return a;
    }

    /**
     * 返回队列的迭代器
     * @return an iterator over the elements in this queue
     */
    public Iterator<E> iterator() {
        return new Itr();
    }
	/**
	*迭代器
	*/
    private final class Itr implements Iterator<E> {
        /**
         * Index (into queue array) of element to be returned by
         * subsequent call to next.
         */
        private int cursor = 0;

        /**
         * Index of element returned by most recent call to next,
         * unless that element came from the forgetMeNot list.
         * Set to -1 if element is deleted by a call to remove.
         */
        private int lastRet = -1;

        /**
         * A queue of elements that were moved from the unvisited portion of
         * the heap into the visited portion as a result of "unlucky" element
         * removals during the iteration.  (Unlucky element removals are those
         * that require a siftup instead of a siftdown.)  We must visit all of
         * the elements in this list to complete the iteration.  We do this
         * after we've completed the "normal" iteration.
         *
         * We expect that most iterations, even those involving removals,
         * will not need to store elements in this field.
         */
        private ArrayDeque<E> forgetMeNot = null;

        /**
         * Element returned by the most recent call to next iff that
         * element was drawn from the forgetMeNot list.
         */
        private E lastRetElt = null;

        /**
         * The modCount value that the iterator believes that the backing
         * Queue should have.  If this expectation is violated, the iterator
         * has detected concurrent modification.
         */
        private int expectedModCount = modCount;

        public boolean hasNext() {
            return cursor < size ||
                (forgetMeNot != null && !forgetMeNot.isEmpty());
        }

        @SuppressWarnings("unchecked")
        public E next() {
            if (expectedModCount != modCount)
                throw new ConcurrentModificationException();
            if (cursor < size)
                return (E) queue[lastRet = cursor++];
            if (forgetMeNot != null) {
                lastRet = -1;
                lastRetElt = forgetMeNot.poll();
                if (lastRetElt != null)
                    return lastRetElt;
            }
            throw new NoSuchElementException();
        }

        public void remove() {
            if (expectedModCount != modCount)
                throw new ConcurrentModificationException();
            if (lastRet != -1) {
                E moved = PriorityQueue.this.removeAt(lastRet);
                lastRet = -1;
                if (moved == null)
                    cursor--;
                else {
                    if (forgetMeNot == null)
                        forgetMeNot = new ArrayDeque<>();
                    forgetMeNot.add(moved);
                }
            } else if (lastRetElt != null) {
                PriorityQueue.this.removeEq(lastRetElt);
                lastRetElt = null;
            } else {
                throw new IllegalStateException();
            }
            expectedModCount = modCount;
        }
    }

    public int size() {
        return size;
    }

    /**
     * Removes all of the elements from this priority queue.
     * The queue will be empty after this call returns.
     */
    public void clear() {
        modCount++;
        for (int i = 0; i < size; i++)
            queue[i] = null;
        size = 0;
    }

    @SuppressWarnings("unchecked")
    public E poll() {
        if (size == 0)
            return null;
        int s = --size;
        modCount++;
        E result = (E) queue[0];
        E x = (E) queue[s];
        queue[s] = null;
        if (s != 0)
            siftDown(0, x);
        return result;
    }

    /**
     * 从队列中remove 第i 个元素
     * 通常,这个方法会保留i-1 一下的元素,包含i-1
     * Removes the ith element from queue.
     *
     * Normally this method leaves the elements at up to i-1,
     * inclusive, untouched.  Under these circumstances, it returns
     * null.  Occasionally, in order to maintain the heap invariant,
     * it must swap a later element of the list with one earlier than
     * i.  Under these circumstances, this method returns the element
     * that was previously at the end of the list and is now at some
     * position before i. This fact is used by iterator.remove so as to
     * avoid missing traversing elements.
     */
    @SuppressWarnings("unchecked")
    private E removeAt(int i) {
        // 防止出现并发操作
        modCount++;
        //队列容量-1
        int s = --size;
        //如果是移除最后一个元素 直接将其赋值为null
        if (s == i) // removed last element
            queue[i] = null;
        else {
            //获取最后一个元素
            E moved = (E) queue[s];
            //释放对象
            queue[s] = null;
            //向下调整
            siftDown(i, moved);
            //如果当前值 与 moved 一致,保持不变 
            if (queue[i] == moved) {
                //向上调整
                siftUp(i, moved);
                //如果不一致,则调整成功
                if (queue[i] != moved)
                    return moved;
            }
        }
        return null;
    }

    /**
     * 在k 位置插入x，并维持最小堆的特性（保持当前节点大于等于其父节点）
     */
    private void siftUp(int k, E x) {
        if (comparator != null)
        	//如果有自定义比较规则
            siftUpUsingComparator(k, x);
        else
            siftUpComparable(k, x);
    }

    @SuppressWarnings("unchecked")
    private void siftUpComparable(int k, E x) {
        Comparable<? super E> key = (Comparable<? super E>) x;
        //k 为待插入节点下标
        //key 为待插入节点
        while (k > 0) {
        	//获取父节点下标
            int parent = (k - 1) >>> 1;
            //获取父节点对象
            Object e = queue[parent];
            //如果当前节点大于等于父节点 直接跳出,跳出后在k插入
            if (key.compareTo((E) e) >= 0)
                break;
            //如果当前节点小于父节点, 将父节点插入k,并重新将父节点下标赋值k,进行下一轮 比较
            queue[k] = e;
            k = parent;
        }
        //将key 赋值给带插入节点
        queue[k] = key;
    }

    @SuppressWarnings("unchecked")
    private void siftUpUsingComparator(int k, E x) {
        while (k > 0) {
            int parent = (k - 1) >>> 1;
            Object e = queue[parent];
            if (comparator.compare(x, (E) e) >= 0)
                break;
            queue[k] = e;
            k = parent;
        }
        queue[k] = x;
    }

    /**
     * Inserts item x at position k, maintaining heap invariant by
     * demoting x down the tree repeatedly until it is less than or
     * equal to its children or is a leaf.
     *
     * @param k the position to fill
     * @param x the item to insert
     */
    private void siftDown(int k, E x) {
        if (comparator != null)
            siftDownUsingComparator(k, x);
        else
            siftDownComparable(k, x);
    }

    @SuppressWarnings("unchecked")
    private void siftDownComparable(int k, E x) {
        Comparable<? super E> key = (Comparable<? super E>)x;
        //获取最后一个元素的父节点作为循环出口
        int half = size >>> 1;        // loop while a non-leaf
        //当k<小于出口
        while (k < half) {
            //获取节点k 的左子节点下标
            int child = (k << 1) + 1; // assume left child is least
            //获取左子节点元素
            Object c = queue[child];
            //获取右子节点下标
            int right = child + 1;
            //right 未越界,获取 c 与 right 中较小的元素,若right 小将其赋值给c
            if (right < size &&
                ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)
                c = queue[child = right];
            //如果待插入节点小于 子节点 直接跳出循环,在位置k赋值key
            if (key.compareTo((E) c) <= 0)
                break;
            //子节点上浮,并将较小节点标记为k 进行下次循环
            queue[k] = c;
            k = child;
        }
        //k 为平衡节点,将元素插入位置key
        queue[k] = key;
    }

    @SuppressWarnings("unchecked")
    private void siftDownUsingComparator(int k, E x) {
        int half = size >>> 1;
        while (k < half) {
            int child = (k << 1) + 1;
            Object c = queue[child];
            int right = child + 1;
            if (right < size &&
                comparator.compare((E) c, (E) queue[right]) > 0)
                c = queue[child = right];
            if (comparator.compare(x, (E) c) <= 0)
                break;
            queue[k] = c;
            k = child;
        }
        queue[k] = x;
    }

    /**
     * Establishes the heap invariant (described above) in the entire tree,
     * assuming nothing about the order of the elements prior to the call.
     */
    @SuppressWarnings("unchecked")
    private void heapify() {
        for (int i = (size >>> 1) - 1; i >= 0; i--)
            siftDown(i, (E) queue[i]);
    }

    /**
     * Returns the comparator used to order the elements in this
     * queue, or {@code null} if this queue is sorted according to
     * the {@linkplain Comparable natural ordering} of its elements.
     *
     * @return the comparator used to order this queue, or
     *         {@code null} if this queue is sorted according to the
     *         natural ordering of its elements
     */
    public Comparator<? super E> comparator() {
        return comparator;
    }

    /**
     * Saves this queue to a stream (that is, serializes it).
     *
     * @serialData The length of the array backing the instance is
     *             emitted (int), followed by all of its elements
     *             (each an {@code Object}) in the proper order.
     * @param s the stream
     */
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException {
        // Write out element count, and any hidden stuff
        s.defaultWriteObject();

        // Write out array length, for compatibility with 1.5 version
        s.writeInt(Math.max(2, size + 1));

        // Write out all elements in the "proper order".
        for (int i = 0; i < size; i++)
            s.writeObject(queue[i]);
    }

    /**
     * Reconstitutes the {@code PriorityQueue} instance from a stream
     * (that is, deserializes it).
     *
     * @param s the stream
     */
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        // Read in size, and any hidden stuff
        s.defaultReadObject();

        // Read in (and discard) array length
        s.readInt();

        queue = new Object[size];

        // Read in all elements.
        for (int i = 0; i < size; i++)
            queue[i] = s.readObject();

        // Elements are guaranteed to be in "proper order", but the
        // spec has never explained what that might be.
        heapify();
    }

    /**
     * Creates a <em><a href="Spliterator.html#binding">late-binding</a></em>
     * and <em>fail-fast</em> {@link Spliterator} over the elements in this
     * queue.
     *
     * <p>The {@code Spliterator} reports {@link Spliterator#SIZED},
     * {@link Spliterator#SUBSIZED}, and {@link Spliterator#NONNULL}.
     * Overriding implementations should document the reporting of additional
     * characteristic values.
     *
     * @return a {@code Spliterator} over the elements in this queue
     * @since 1.8
     */
    public final Spliterator<E> spliterator() {
        return new PriorityQueueSpliterator<E>(this, 0, -1, 0);
    }

    static final class PriorityQueueSpliterator<E> implements Spliterator<E> {
        /*
         * This is very similar to ArrayList Spliterator, except for
         * extra null checks.
         */
        private final PriorityQueue<E> pq;
        private int index;            // current index, modified on advance/split
        private int fence;            // -1 until first use
        private int expectedModCount; // initialized when fence set

        /** Creates new spliterator covering the given range */
        PriorityQueueSpliterator(PriorityQueue<E> pq, int origin, int fence,
                             int expectedModCount) {
            this.pq = pq;
            this.index = origin;
            this.fence = fence;
            this.expectedModCount = expectedModCount;
        }

        private int getFence() { // initialize fence to size on first use
            int hi;
            if ((hi = fence) < 0) {
                expectedModCount = pq.modCount;
                hi = fence = pq.size;
            }
            return hi;
        }

        public PriorityQueueSpliterator<E> trySplit() {
            int hi = getFence(), lo = index, mid = (lo + hi) >>> 1;
            return (lo >= mid) ? null :
                new PriorityQueueSpliterator<E>(pq, lo, index = mid,
                                                expectedModCount);
        }

        @SuppressWarnings("unchecked")
        public void forEachRemaining(Consumer<? super E> action) {
            int i, hi, mc; // hoist accesses and checks from loop
            PriorityQueue<E> q; Object[] a;
            if (action == null)
                throw new NullPointerException();
            if ((q = pq) != null && (a = q.queue) != null) {
                if ((hi = fence) < 0) {
                    mc = q.modCount;
                    hi = q.size;
                }
                else
                    mc = expectedModCount;
                if ((i = index) >= 0 && (index = hi) <= a.length) {
                    for (E e;; ++i) {
                        if (i < hi) {
                            if ((e = (E) a[i]) == null) // must be CME
                                break;
                            action.accept(e);
                        }
                        else if (q.modCount != mc)
                            break;
                        else
                            return;
                    }
                }
            }
            throw new ConcurrentModificationException();
        }

        public boolean tryAdvance(Consumer<? super E> action) {
            if (action == null)
                throw new NullPointerException();
            int hi = getFence(), lo = index;
            if (lo >= 0 && lo < hi) {
                index = lo + 1;
                @SuppressWarnings("unchecked") E e = (E)pq.queue[lo];
                if (e == null)
                    throw new ConcurrentModificationException();
                action.accept(e);
                if (pq.modCount != expectedModCount)
                    throw new ConcurrentModificationException();
                return true;
            }
            return false;
        }

        public long estimateSize() {
            return (long) (getFence() - index);
        }

        public int characteristics() {
            return Spliterator.SIZED | Spliterator.SUBSIZED | Spliterator.NONNULL;
        }
    }
}

```

