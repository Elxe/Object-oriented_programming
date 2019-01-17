# 2.功能分析与建模
---

## 2.1不可变集合（Immutable_collections）

### 2.1.1简介

Immutable collections，顾名思义就是说集合是不可被修改的。集合的数据项是在创建的时候提供，并且在整个生命周期中都不可改变。

### 2.1.2优势

1. 不受信任的库可以安全使用。
2. 线程安全：可以被许多线程使用，没有竞争条件的风险不可变性，可以节省时间和空间。
3. 所有不可变集合的实现都比其可变的实现占用较少的资源，效率也较高。（可变集合相较于不可变集合有更多的开销，包括并发修改检查，哈希表中的额外空间等）
4. 可以用作常量使用。 

### 2.1.3创建方式

* 使用copyOf方法创建，例如：`ImmutableSet.copyOf(set);`
* 使用of方法，例如：`ImmutableSet.of("a", "b", "c");`
* 使用Builder，例如：`public static final ImmutableSet<Color> GOOGLE_COLORS = ImmutableSet.<Color>builder().addAll(WEBSAFE_COLORS).add(new Color(0, 191, 255)).build();`

### 2.1.4示例代码

```java
public static final ImmutableSet<String> COLOR_NAMES = ImmutableSet.of (
    "red",    
    "orange",    
    "yellow",    
    "green",    
    "blue",    
    "purple");
class Foo {    
    final ImmutableSet<Bar> bars;    
    Foo(Set<Bar> bars) {        
        this.bars = ImmutableSet.copyOf(bars);    
        }
}
```

上面的代码，`ImmutableList.copyOf(bars)`会智能的返回`foobar.asList()`。

`ImmutableSet.copyOf(ImmutableCollection)`会在以下几种情况下避免线性拷贝：

* 在时间常数之内相关的数据结构可用。例如`ImmutableSet.copyOf(ImmutableList)`在时间常数内就无法完成。
* 不会造成内存泄漏，例如，有一个`ImmutableList<String>`的hugeList，然后调用`ImmutableList.copyOf(hugeList.subList(0, 10))`，拷贝会立即执行，是为了避免持有不需要的hugeList的引用。
* 不会改变语义，`ImmutableSet.copyOf(myImmutableSortedSet)`会执行一个拷贝，因为ImmutableSet使用的`hashCode()`和`equals()`方法与基于Comparator行为的ImmutableSortedSet具有不同的语义。

这些都是为了在防御式编程风格中保持最小的性能开销。

## 2.2多集(Multiset)

### 2.2.1简介

Multiset，支持添加多个元素。维基百科对multiset的数学定义如下：一般集合的概念是允许相同的元素出现多次，而multiset，类似于集合但与元组相反，是顺序无关的，例如：`{a, a, b}`与`{a, b, a}`这两个multiset是相等的。

### 2.2.2优势

普通的集合类在进行相同元素个数统计时，操作显得过于繁琐。Multiset为仅关注元素个数的集合操作提供非常便利的操作方式。Multiset类似于无序的`ArrayList<E>` ，也类似于包含元素与计数的`Map<E, Integer>` 。

另外，`Multiset`与`Collection`接口是完全一致的（除了极少数的情况），所以可以很容易的用Mutiliset代替Collection。

### 2.2.3实现

Guava有多个Multiset实现：

| **Map** | **对应的Multiset** |
| :--- | :--- |
| HasMap | HashMultiset |
| TreeMap | TreeMultiset |
| LinkedHashMap | LinkedHashMultiset |
| ConcurrentHashMap | ConcurrentHashMultiset |
| ImmutableMap | ImmutableMultiset |

### 2.2.4拓展

`SortedMultiset`是在`Multiset`接口基础上的一个变体，它支持获取指定范围的子`Multiset`。

## 2.3Multimap

###  2.3.1 简介

Multimap是将元素与任意多个值相关联的一般方法。

### 2.3.2优势

Guava的`MultiMap`框架可以让键到多值的映射更加简单。

### 2.3.3创建方式

值得一提的是一般我们很少会直接使用`MultiMap`接口，更经常的是使用`ListMultiMap`（映射key到List）或`SetMultimap`（映射key到Set）。以下给出这两个类的创建方式：

```java
ListMultimap<String, Integer> treeListMultimap =
 MultimapBuilder.treeKeys().arrayListValues().build();

SetMultimap<Integer, MyEnum> hashEnumMultimap =
 MultimapBuilder.hashKeys().enumSetValues(MyEnum.class).build();
```

### 2.3.4使用情景

从概念上讲，有两种方法可以考虑Multimap：

* 作为从单个键到单个值的映射集合：
  ```
    a -> 1
    a -> 2
    a -> 4
    b -> 3
    c -> 5
  ```
* 或者作为从唯一键到值集合的映射：
  ```
    a -> [1, 2, 4]
    b -> [3]
    c -> [5]
  ```

## 2.4BiMap

### 2.4.1简介

BiMap<K, V>是一个值是唯一的双射，允许使用inverse()反转查看BiMap<V, K>。

传统的键值双向映射是维护两个Map并保持它们的同步，这样很容易导致bug的出现，并且当value已经在map中的时候会让人感到困惑，例如：

```java
Map<String, Integer> nameToId = Maps.newHashMap();
Map<Integer, String> idToName = Maps.newHashMap();

nameToId.put("Bob", 42);
idToName.put(42, "Bob");
//如果Bob或42已经存在Map中该怎么办？
//如果忘记同步了会出现奇怪的bug...
```

`BiMap.put(key, value)`会抛出`IllegalArgumentException`如果映射key到一个已存在的value。如果要删除与此value关联的key，使用`BiMap.forcePut(key, value)`。

```java
BiMap<String, Integer> userId = HashBiMap.create();	
...
String userForId = userId.inverse().get(id);
```

### 2.4.2实现

支持HashBiMap、ImmutableBiMap、EnumBiMap、EnumHashBiMap。

## 2.5Table

### 2.5.1简介

当想要同时以多个key索引时，会使用到`Map<FirstName, Map<LastName, Person>>`这种丑陋的结构。Guava提供了一个新的集合类型 - Table，适用于这种基于“行列”的情景。它有一系列的视图可供使用：

```java
Table<Vertex, Vertex, Double> weightedGraph = HashBasedTable.create();
weightedGraph.put(v1, v2, 4);
weightedGraph.put(v1, v3, 20);
weightedGraph.put(v2, v3, 5);

weightedGraph.row(v1); //返回一个映射（v2->4, v3->20）
weightedGraph.column(v3); //返回一个映射(v1->20, v2->5)
```

* `rowMap()`，将`Table<R, C, V>`作为`Map<R, Map<C, V>>`视图查看，类似的，`rowKeySet()`返回一个`Set<R>`
* `row(r)`返回一个非null的`Map<C, V>`，对这个Map的操作将会影响到关联的Table
* 类似的基于列的方法：`columnMap()`, `columnKeySet()`, `column(c)`（基于列的访问效率要低于基于行的访问）
* `cellSet()`将`Table.Cell<R, C, V>`的集合作为Table的视图返回   

以下是Table的实现：

* HashBasedTable，由`HashMap<R, HashMap<C, V>>`实现
* TreeBasedTable，由`TreeMap<R, TreeMap<C, V>>`实现
* ImmutableTable，由`ImmutableMap<R, ImmutableMap<C, V>>`（ImmutableTable是为稀疏和密集数据集优化的实现）
* ArrayTable，全部的行列需要在构造的时候指定，当该Table是密集型的时候为了内存和速度的效率，它由二维数组实现的。ArrayTable的实现与其他的实现有些不同，具体参见Javadoc。

## 2.6RangeSet

### 2.6.1简介 
RangeSet描述了一组断开的、非空的范围集（并集）。当把一个范围添加到一个范围集的时候，所有可连接的范围集（交集）都会合并为一个范围，空集将被忽略。例如：

```java
RangeSet<Integer> rangeSet = TreeRangeSet.create();
rangeSet.add(Range.closed(1, 10)); //{[1, 10]}
rangeSet.add(Range.closedOpen(11, 15)); //断开的范围集，{[1, 10], [11, 15)}
rangeSet.add(Range.closedOpen(15, 20)); //可连接的范围（有交集），{[1, 10], [11, 20)}
rangeSet.add(Range.openClosed(0, 0)); //空集，{[1, 10], [11, 20)}
rangeSet。remove(Range.open(5, 10)); //[1, 10]区间被分隔了，{[1, 5], [10, 10], [11, 20]}
```

### 2.6.2视图

RangeSet实现支持很多范围的视图，包括：

* `complement();`
* `sunRangeSet(Range<C>); //返回该范围集和指定的范围的交集`
* `asRanges();`
* `asSet(DiscreteDomain<C>);`

## 2.7RangeMap

### 2.7.1简介
RangeMap是一组范围集到值的映射，不像RangeSet，RangeMap从来不会合并相邻的映射，即使相邻的范围集映射到相同的值。例如：

```java
RangeMap<Integer, String> rangeMap = TreeRangeMap.create();
rangeMap.put(Range.closed(1, 10), "foo"); //{[1, 10] -> "foo"}
rangeMap.put(Range.open(3, 6), "bar"); //{[1, 3] - > "foo", (3, 6) -> "bar", [6, 10] -> "foo"}
rangeMap.put(Range.open(10, 20), "foo"); //{[1, 3] - > "foo", (3, 6) -> "bar", [6, 10] -> "foo", (10, 20) -> "foo"}
rangeMap.remove(Range.closed(5, 11)); //{[1, 3] - > "foo", (3, 5) -> "bar", [11, 20) -> "foo"}
```

### 2.7.2视图

RangeMap提供了两种视图：

* `asMapOfRanges()`：将RangeMap作为`Map<Range<K>, V>`视图返回，支持迭代该RangeMap。
* `subRangeMap(Range<K>)`，返回该RangeMap与指定Range的交集视图，传统的headMap, subMap, tailMap操作就是建立在此基础上。
<br> 
<br> 
 

**引用：**
> [guava中文文档](https://github.com/kwf2030/guava-userguide-cn/tree/master/集合)
> [guava官方使用手册](https://github.com/google/guava/wiki/ImmutableCollectionsExplained)


