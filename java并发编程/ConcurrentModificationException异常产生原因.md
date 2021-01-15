ConcurrentModificationException是基于java集合中的 **快速失败（fail-fast）** 机制产生的，在使用迭代器遍历一个集合对象时，如果遍历过程中对集合对象的内容进行了增删改，就会抛出该异常。
快速失败机制使得java的集合类不能在多线程下并发修改，也不能在迭代过程中被修改。

#### 1. 抛异常实例

使用迭代器遍历list，同时使用list.remove方法删除元素，会抛ConcurrentModificationException异常：

```java
public void tryRemove(List<String> strings) {
    Iterator<String> it = strings.iterator();
    while(it.hasNext()) {
        String str = it.next();
        if(str.equals("remove") {
            strings.remove(str);
        }
    }
}
```

#### 2. 抛异常的原因

抛异常是在next方法中，该方法共有两处抛异常，上面的实例中，是在next方法的第一处抛的异常。

```java
public E next() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
    int i = cursor;
    if (i >= limit)
        throw new NoSuchElementException();
    Object[] elementData = ArrayList.this.elementData;
    if (i >= elementData.length)
        throw new ConcurrentModificationException();
    cursor = i + 1;
    return (E) elementData[lastRet = i];
}
```

那么这个modCount的作用是什么呢？
modCount是在AbstractList抽象类中定义，且声明为```protected transient```，该变量实际就是个修改次数的计数标志，以ArrayList为例，会在ArrayList的以下方法中执行```modCount++```:

```java
public void trimToSize() { ... }
private void ensureExplicitCapacity(int minCapacity) { ... }
public E remove(int index) { ... }
private void fastRemove(int index) { ... }
public void clear() { ... }
protected void removeRange(int fromIndex, int toIndex) { ... }
public boolean removeIf(Predicate<? super E> filter) { ... }
public void replaceAll(UnaryOperator<E> operator) { ... }
public void sort(Comparator<? super E> c) { ... }
```

add方法内部会调用到ensureExplicitCapacity方法，也间接调用了modCount++。
这些方法的共同点：变更了list的数组长度或移动了list内部的元素。只要list集合内部数组结构上发生了变更，modCount就会自增1。
所以上面的例子中，expectedModCount值记录的是调用iterator方法时的modCount值，而调用过一次list.remove之后，modCount自增1，也就与expectedModCount不相等了，所以就抛出异常，表明在迭代过程中list被修改了而快速失败。
因此在使用迭代器遍历删除集合元素时，应使用迭代器的remove方法，其内部会更新expectedModCount及cursor，以避免调用next方法时抛异常。

```java
private class Itr implements Iterator<E> {
    ......
    public void remove() {
        if (lastRet < 0)
           throw new IllegalStateException();
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();

        try {
            ArrayList.this.remove(lastRet);
            cursor = lastRet;//更新cursor，避免next方法中的第二处抛异常
            lastRet = -1;
            expectedModCount = modCount;//调用remove后更新expectedModCount
            limit--;//更新列表长度
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }
    ......
}
```

#### 3. 如何避免ConcurrentModificationException异常

modCount只在集合的内部迭代器中使用，所以规避该异常只需在使用迭代器时考虑迭代器内部的expectedModCount与list的modCount值是否相等，游标cursor是否可能大于数组长度。
基于此思考归纳出以下几点可能抛该异常的情景：

- 迭代器遍历过程中，调用了集合的remove或add等改变了集合结构的方法；
- foreach循环遍历集合，实际上隐式调用了迭代器遍历，同样调用集合的remove或add等方法会抛异常；
- 多线程环境下，迭代器遍历+iterator.remove也可能导致异常，因此多线程环境下建议使用并发集合或做好线程间的同步。
