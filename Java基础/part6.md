# 理解泛型

1. 理解泛型通配符

   * ```<? extends Fruit>``` ：协变，生产者，提供 ```Fruit``` 或者 ```Fruit``` 的子类。```List<？extends Fruit>```不能 add 任何元素，可以 get 元素出来。
   * ```<? super Fruit>```：逆变，消费者，接收 Fruit 或者 Fruit 的子类，```List<? super Fruit>```可以添加 ```Fruit``` 以及 ```Fruit``` 的子类元素。
   * ```<?>```：无类型，```List<?>```不能添加任何元素。

2. 可不可以创建泛型类数组？为什么？

   不可以，因为泛型擦除会导致泛型类数组类型不安全。

3. 什么是泛型擦除？

   泛型信息在编译的时候会被擦除成它的边界，JVM里面是没有泛型相关的信息的。

   * ```<T>``` **&rarr;** ```T``` 被擦除成 ```Object```
   * ```<T extends Fruit>``` **&rarr;** ```T``` 被擦除成 ```Fruit```
   * ```<T super Fruit>``` **&rarr;** ```T``` 被擦除成 ```Fruit```

   

   