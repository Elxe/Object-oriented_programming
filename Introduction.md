# 1. Guava简介

---

## 1.1Guava是什么？

Guava 是 Google 提供的一组 Java 核心类库，包括新的集合框架\(比如 multimap 和 multiset\), 不可变集合 \(immutable collections\), 图形库\(graph library\), 函数 \(functional types\), 内存中缓存技术\(in-memory cache\), 以及用于并发的 API, I/O, 哈希算法\(hashing\), 原语\(primitives\), 反射\(reflection\), 字符串处理\(string processing\)等。

## 1.2Guava集合是什么？

其中集合组件的创建和体系结构部分是由JDK1.5中引入的泛型驱动的。尽管泛型提高了程序员的工作效率，但标准JCF并没有提供足够的功能，而且它的补充Apache Commons Collections没有采用泛型来保持向后兼容性。这一事实促使两位工程师Kevin Bourrillion和Jared Levy开发了JCF扩展，后者提供了额外的泛型类，如多集，多图，bimaps以及不可变集合。

