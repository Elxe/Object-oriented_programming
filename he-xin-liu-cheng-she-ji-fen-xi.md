# 3.核心流程设计分析
---

（本来这部分写的内容挺多的，但是文件被还原了，我很难受）

由于guava的集合部分代码量过于庞大，因此下文将仅对**ImmutableSet**进行分析。

`ImmutableSet`思想很简单，就是对外不提供修改的接口，只将创建的方法`public`。但是guava集合却用了30个文件，401KB的文件实现这一个功能。

类的继承关系为：
![](/assets/13F9.png)
![](/assets/L.png)

通过上一章我们知道ImmutableSet总共有如下三种创建方式：

* 使用copyOf方法创建，例如：`ImmutableSet.copyOf(set);`
* 使用of方法，例如：`ImmutableSet.of("a", "b", "c");`
* 使用Builder，例如：`public static final ImmutableSet<Color> GOOGLE_COLORS = ImmutableSet.<Color>builder().addAll(WEBSAFE_COLORS).add(new Color(0, 191, 255)).build();`

接下来我们来探究一下这3种方法究竟是怎么运作的。

## 3.1of创建方法流程

首先，我们看这段代码：

```java
public abstract class ImmutableSet<E> extends ImmutableCollection<E> implements Set<E> {
    ...
    //以上省略6个元素以下的of创建方法
    public static <E> ImmutableSet<E> of(E e1, E e2, E e3, E e4, E e5, E e6, E... others) {
        checkArgument(
            others.length <= Integer.MAX_VALUE - 6,
            "the total number of elements must fit in an int");
        final int paramCount = 6;
        Object[] elements = new Object[paramCount + others.length];
        elements[0] = e1;
        elements[1] = e2;
        elements[2] = e3;
        elements[3] = e4;
        elements[4] = e5;
        elements[5] = e6;
        System.arraycopy(others, 0, elements, paramCount, others.length);
        return construct(elements.length, elements);
  }
  ...
}
```

可以看到ImmutableSet继承了ImmutableCollection，ImmutableCollection是谷歌guava提供的一个抽象类，而ImmutableList、ImmutableMap等类都继承了ImmutableCollecion这个抽象类。之后会具体说明ImmutableSet用到的ImmutableCollection中的方法。

of方法很容易理解，如果参数个数大于6个，需要先调用arraycopy方法进行处理，6个参数以下就直接调用创建方法。

```java
// 0个元素
  public static <E> ImmutableSet<E> of() {
    return (ImmutableSet<E>) RegularImmutableSet.EMPTY;
  }
// 单个元素
  public static <E> ImmutableSet<E> of(E element) {
    return new SingletonImmutableSet<E>(element);
  }
...
  private static <E> ImmutableSet<E> construct(int n, Object... elements) {
    switch (n) {
      case 0:
        return of();
      case 1:
        @SuppressWarnings("unchecked") // safe; elements contains only E's
        E elem = (E) elements[0];
        return of(elem);
      default:
        SetBuilderImpl<E> builder =
            new RegularSetBuilderImpl<E>(ImmutableCollection.Builder.DEFAULT_INITIAL_CAPACITY);
        for (int i = 0; i < n; i++) {
          @SuppressWarnings("unchecked")
          E e = (E) checkNotNull(elements[i]);
          builder = builder.add(e);
        }
        return builder.review().build();
    }
  }
```

如果元素的长度等于0就调用RegularImmutableSet.EMPTY方法返回一个空集：

```java
final class RegularImmutableSet<E> extends ImmutableSet<E> {
  //集合为空调用下面RegularImmutableSet<>(new Object[0], 0, null, 0)创建
  static final RegularImmutableSet<Object> EMPTY =
      new RegularImmutableSet<>(new Object[0], 0, null, 0);

  private final transient Object[] elements;
  // the same elements in hashed positions (plus nulls)
  @VisibleForTesting final transient Object[] table;
  // 'and' with an int to get a valid table index.
  private final transient int mask;
  private final transient int hashCode;

  RegularImmutableSet(Object[] elements, int hashCode, Object[] table, int mask) {
    this.elements = elements;
    this.table = table;
    this.mask = mask;
    this.hashCode = hashCode;
  }
  ...
}
```

元素个数等于1调用SingletonImmutableSet创建单个元素的集合：

```java
final class SingletonImmutableSet<E> extends ImmutableSet<E> {

  final transient E element;
  @LazyInit private transient int cachedHashCode;

  SingletonImmutableSet(E element) {
    this.element = Preconditions.checkNotNull(element);
  }

  SingletonImmutableSet(E element, int hashCode) {
    // Guaranteed to be non-null by the presence of the pre-computed hash code.
    this.element = element;
    cachedHashCode = hashCode;
  }
  ...
}
```

元素个数大于1则调用`RegularSetBuilderImpl`类的review方法进行创建：

```java
private static final class RegularSetBuilderImpl<E> extends SetBuilderImpl<E> {
    private Object[] hashTable;
    private int maxRunBeforeFallback;
    private int expandTableThreshold;
    private int hashCode;

    RegularSetBuilderImpl(int expectedCapacity) {
      super(expectedCapacity);
      int tableSize = chooseTableSize(expectedCapacity);
      this.hashTable = new Object[tableSize];
      this.maxRunBeforeFallback = maxRunBeforeFallback(tableSize);
      this.expandTableThreshold = (int) (DESIRED_LOAD_FACTOR * tableSize);
    }

    RegularSetBuilderImpl(RegularSetBuilderImpl<E> toCopy) {
      super(toCopy);
      this.hashTable = Arrays.copyOf(toCopy.hashTable, toCopy.hashTable.length);
      this.maxRunBeforeFallback = toCopy.maxRunBeforeFallback;
      this.expandTableThreshold = toCopy.expandTableThreshold;
      this.hashCode = toCopy.hashCode;
    }

    void ensureTableCapacity(int minCapacity) {
      if (minCapacity > expandTableThreshold && hashTable.length < MAX_TABLE_SIZE) {
        int newTableSize = hashTable.length * 2;
        hashTable = rebuildHashTable(newTableSize, dedupedElements, distinct);
        maxRunBeforeFallback = maxRunBeforeFallback(newTableSize);
        expandTableThreshold = (int) (DESIRED_LOAD_FACTOR * newTableSize);
      }
    }

    @Override
    SetBuilderImpl<E> add(E e) {
      checkNotNull(e);
      int eHash = e.hashCode();
      int i0 = Hashing.smear(eHash);
      int mask = hashTable.length - 1;
      for (int i = i0; i - i0 < maxRunBeforeFallback; i++) {
        int index = i & mask;
        Object tableEntry = hashTable[index];
        if (tableEntry == null) {
          addDedupedElement(e);
          hashTable[index] = e;
          hashCode += eHash;
          ensureTableCapacity(distinct); // rebuilds table if necessary
          return this;
        } else if (tableEntry.equals(e)) { // not a new element, ignore
          return this;
        }
      }
      // we fell out of the loop due to a long run; fall back to JDK impl
      return new JdkBackedSetBuilderImpl<E>(this).add(e);
    }

    @Override
    SetBuilderImpl<E> copy() {
      return new RegularSetBuilderImpl<E>(this);
    }

    @Override
    SetBuilderImpl<E> review() {
      int targetTableSize = chooseTableSize(distinct);
      if (targetTableSize * 2 < hashTable.length) {
        hashTable = rebuildHashTable(targetTableSize, dedupedElements, distinct);
      }
      return hashFloodingDetected(hashTable) ? new JdkBackedSetBuilderImpl<E>(this) : this;
    }

    @Override
    ImmutableSet<E> build() {
      switch (distinct) {
        case 0:
          return of();
        case 1:
          return of(dedupedElements[0]);
        default:
          Object[] elements =
              (distinct == dedupedElements.length)
                  ? dedupedElements
                  : Arrays.copyOf(dedupedElements, distinct);
          return new RegularImmutableSet<E>(elements, hashCode, hashTable, hashTable.length - 1);
      }
    }
  }
```

`ImmutableSet.Builder`的内部的默认实现，创建一个开放地址的哈希表和重复删除元素，因此它只分配`O(max(distinct, expectedCapacity))`而不是`O(calls to add)`。

此实现尝试检测哈希flood，如果已识别，则回退到`JdkBackedSetBuilderImpl`。

## 3.2copyOf创建方法流程

copyOf有如下4种实现方式：

1. 按第一次出现在源`Collection`中的顺序返回包含每个`elements`的不可变集合，并减去重复项。可以看到这个方法先对elements的类型进行了判断，
   ```java
   public static <E> ImmutableSet<E> copyOf(Collection<? extends E> elements) {
    if (elements instanceof ImmutableSet && !(elements instanceof SortedSet)) {
      @SuppressWarnings("unchecked") // all supported methods are covariant
      ImmutableSet<E> set = (ImmutableSet<E>) elements;
      if (!set.isPartialView()) {
        return set;
      }
    } else if (elements instanceof EnumSet) {
      return copyOfEnumSet((EnumSet) elements);
    }
    Object[] array = elements.toArray();
    return construct(array.length, array);
   }
   ```
2. 按第一次出现在源`Iterable`中的顺序返回包含每个`elements`的不可变集合，并减去重复项。并调用`instanceof`方法检查`elements`是否为`Collection`的子类的实例，如果是便调用`copyOf((Collection<? extends E>) elements)`方法进行创建，如果不是则调用`copyOf(elements.iterator())`进行创建。
```java
public static &lt;E&gt; ImmutableSet&lt;E&gt; copyOf(Iterable<extends E>; elements) {
 return (elements instanceof Collection)
     ? copyOf((Collection<? extends E>) elements)
     : copyOf(elements.iterator());
}
```

3. 按第一次出现在源`Iterator`中的顺序返回包含每个`elements`的不可变集合，并减去重复项。
   ```java
   public static <E> ImmutableSet<E> copyOf(Iterator<? extends E> elements) {
    // We special-case for 0 or 1 elements, but anything further is madness.
    if (!elements.hasNext()) {
      return of();
    }
    E first = elements.next();
    if (!elements.hasNext()) {
      return of(first);
    } else {
      return new ImmutableSet.Builder<E>().add(first).addAll(elements).build();
    }
   }
   ```
4. 按第一次出现在源`array`中的顺序返回包含每个`elements`的不可变集合，并减去重复项。

   ```java
   public static <E> ImmutableSet<E> copyOf(E[] elements) {
    switch (elements.length) {
      case 0:
        return of();
      case 1:
        return of(elements[0]);
      default:
        return construct(elements.length, elements.clone());
    }
   }
   ```

## 3.3Builder创建方法
   Builder方法创建的一般方式是将所有的元素通过addall添加到Builer，再通过private SetBuilderImpl<E> impl类的build方法创建。
   ```java
   public static class Builder<E> extends ImmutableCollection.Builder<E> {
    private SetBuilderImpl<E> impl;
    boolean forceCopy;

    public Builder() {
      this(DEFAULT_INITIAL_CAPACITY);
    }

    Builder(int capacity) {
      impl = new RegularSetBuilderImpl<E>(capacity);
    }

    Builder(@SuppressWarnings("unused") boolean subclass) {
      this.impl = null; // unused
    }

    @VisibleForTesting
    void forceJdk() {
      this.impl = new JdkBackedSetBuilderImpl<E>(impl);
    }

    final void copyIfNecessary() {
      if (forceCopy) {
        copy();
        forceCopy = false;
      }
    }

    void copy() {
      impl = impl.copy();
    }

    @Override
    @CanIgnoreReturnValue
    public Builder<E> add(E element) {
      checkNotNull(element);
      copyIfNecessary();
      impl = impl.add(element);
      return this;
    }

    @Override
    @CanIgnoreReturnValue
    public Builder<E> add(E... elements) {
      super.add(elements);
      return this;
    }

    //将每个元素添加到ImmutableSet，忽略重复元素（仅添加第一个重复元素）。
    @CanIgnoreReturnValue
    public Builder<E> addAll(Iterable<? extends E> elements) {
      super.addAll(elements);
      return this;
    }

    @Override
    @CanIgnoreReturnValue
    public Builder<E> addAll(Iterator<? extends E> elements) {
      super.addAll(elements);
      return this;
    }

    Builder<E> combine(Builder<E> other) {
      copyIfNecessary();
      this.impl = this.impl.combine(other.impl);
      return this;
    }

    @Override
    public ImmutableSet<E> build() {
      forceCopy = true;
      impl = impl.review();
      return impl.build();
    }
   }
   ```

其实`ImmutableSet`的原理很简单，即不对外提供修改的接口，只提供的创建的类，而将修改的类`private`、隐藏，就能达到这个目的。大篇幅的代码是为了更好的跟现有的java库进行对接，提高`ImmutableSet`的通用性。

