# 写在前面
---

在本文主要讨论**谷歌Guava中的集合功能**的设计与实现，从**功能分析与建模**、**核心流程设计分析**、**高级设计意图分析**等角度关注源码中的特定功能，分析源码。

此项目为**国科大面向对象程序设计课**的大作业，之前在选项目的时候被老师的难度分类给骗了，本来想选一个比较简单的项目，但是却选到一个代码量将近80,000行的项目，对于我这种没有学过Java的人来说实在是过于困难，所以可能本文表达的观点有一定的局限性。

此文在完成课程作业的同时，也希望能对大家有所帮助。如果大家对此文有任何的意见或者是建议，欢迎交流。

作者：Elxe

E-mail: elxe\_in\_ucas@sohu.com

# Summary

* [写在前面](README.md)
* [Guava介绍](Introduction.md)
  * [Guava是什么？](Introduction.md##1.1Guava是什么？)
  * [Guava集合是什么？](Introduction.md##1.2Guava集合是什么？)
* [功能分析与建模](gong-neng-fen-xi-yu-jian-mo.md)
  * [不可变集合（Immutable collections）](gong-neng-fen-xi-yu-jian-mo.md##2.1不可变集合（Immutable_collections）)
  * [多集 (Multiset)](gong-neng-fen-xi-yu-jian-mo.md##2.2多集（Multiset)
  * [Multimap](gong-neng-fen-xi-yu-jian-mo.md##2.3Multimap)
  * [BiMap](gong-neng-fen-xi-yu-jian-mo.md##2.4BiMap)
  * [Table](gong-neng-fen-xi-yu-jian-mo.md##2.5Table)
  * [RangeSet](gong-neng-fen-xi-yu-jian-mo.md##2.6RangeSet)
  * [RangeMap](gong-neng-fen-xi-yu-jian-mo.md##2.7RangeMap)
* [核心流程设计分析](he-xin-liu-cheng-she-ji-fen-xi.md)
  * [of创建方法流程](he-xin-liu-cheng-she-ji-fen-xi.md##3.1of创建方法流程)
  * [copyOf创建方法流程](he-xin-liu-cheng-she-ji-fen-xi.md##3.2copyOf创建方法流程)
  * [Builder创建方法](he-xin-liu-cheng-she-ji-fen-xi.md##3.3Builder创建方法)
* [高级设计意图分析](gao-ji-she-ji-yi-tu-fen-xi.md)
  * [抽象工厂](gao-ji-she-ji-yi-tu-fen-xi.md##4.1抽象工厂)
  * [建造者模式（Builder）](gao-ji-she-ji-yi-tu-fen-xi.md##4.2建造者模式（Builder）)
  * [适配器模式](gao-ji-she-ji-yi-tu-fen-xi.md##4.2适配器模式)
* [写在最后](xie-zai-zui-hou.md)