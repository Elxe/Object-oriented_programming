#4.高级设计意图分析
---

此章节分析一下guava集合部分的高级设计意图。

##4.1抽象工厂

首先介绍一下guava集合里使用的抽象工厂：

java.util.Iterator接口是根抽象产品，下面有AsMap、Wrapped等抽象产品，还有AsMapIterator、WrappedIterator等作为具体产品。

使用不同的具体工厂类中的iterator方法能得到不同的具体产品的实例。

```java
public interface Iterator<E> {
  boolean hasNext();
  E next();
  void remove();
}
```

```java
    class AsMapIterator implements Iterator<Entry<K, Collection<V>>> {
      final Iterator<Entry<K, Collection<V>>> delegateIterator = submap.entrySet().iterator();
      @Nullable Collection<V> collection;

      @Override
      public boolean hasNext() {
        return delegateIterator.hasNext();
      }

      @Override
      public Entry<K, Collection<V>> next() {
        Entry<K, Collection<V>> entry = delegateIterator.next();
        collection = entry.getValue();
        return wrapEntry(entry);
      }

      @Override
      public void remove() {
        checkRemove(collection != null);
        delegateIterator.remove();
        totalSize -= collection.size();
        collection.clear();
        collection = null;
      }
    }
```

```java
    class WrappedIterator implements Iterator<V> {
      final Iterator<V> delegateIterator;
      final Collection<V> originalDelegate = delegate;

      ...

      @Override
      public boolean hasNext() {
        validateIterator();
        return delegateIterator.hasNext();
      }

      @Override
      public V next() {
        validateIterator();
        return delegateIterator.next();
      }

      @Override
      public void remove() {
        delegateIterator.remove();
        totalSize--;
        removeIfEmpty();
      }

      Iterator<V> getDelegateIterator() {
        validateIterator();
        return delegateIterator;
      }
    }
```

##4.2建造者模式（Builder）

关于建造者模式其实在上一章介绍创建流程时已经有所涉及，这里再进行深入分析。

```java
  public abstract static class Builder<E> {
    static final int DEFAULT_INITIAL_CAPACITY = 4;

    static int expandedCapacity(int oldCapacity, int minCapacity) {
      if (minCapacity < 0) {
        throw new AssertionError("cannot store more than MAX_VALUE elements");
      }
      // careful of overflow!
      int newCapacity = oldCapacity + (oldCapacity >> 1) + 1;
      if (newCapacity < minCapacity) {
        newCapacity = Integer.highestOneBit(minCapacity - 1) << 1;
      }
      if (newCapacity < 0) {
        newCapacity = Integer.MAX_VALUE;
        // guaranteed to be >= newCapacity
      }
      return newCapacity;
    }

    Builder() {}

    @CanIgnoreReturnValue
    public abstract Builder<E> add(E element);

    @CanIgnoreReturnValue
    public Builder<E> add(E... elements) {
      for (E element : elements) {
        add(element);
      }
      return this;
    }

    @CanIgnoreReturnValue
    public Builder<E> addAll(Iterable<? extends E> elements) {
      for (E element : elements) {
        add(element);
      }
      return this;
    }

    @CanIgnoreReturnValue
    public Builder<E> addAll(Iterator<? extends E> elements) {
      while (elements.hasNext()) {
        add(elements.next());
      }
      return this;
    }
    
    public abstract ImmutableCollection<E> build();
  }
```
在集合元素个数不确定的情况下，builder设计模式很好地解决了创建时的尴尬，并使代码变得更加简洁。


##4.3适配器模式

![](/assets/13F9.png)
其中`public abstract class ImmutableCollection<E> extends AbstractCollection<E> implements Serializable`的`ImmutableCollection`继承`AbstractCollection`类，并实现接口`Serializable`；`public abstract class ImmutableSet<E> extends ImmutableCollection<E> implements Set<E>`的`ImmutableSet`继承`ImmutableCollection`类，并实现接口`Set`。

