# JVM运行时数据区

1. JVM虚拟机规范定义的运行时数据区（JVM内存模型）？

   ![jvm](https://thismj.nos-eastchina1.126.net/image/gitbook_jvm_memory_model.jpg)

   * 程序计数器
   * Java虚拟机栈 - 栈帧Frame - （局部变量表Local Variables+操作数栈Operand Stack+动态链接Dynamic Linking+方法出口）
   * 本地方法栈
   * 堆
   * 方法区（HotSpot的实现，1.7以前称之为永久代PermGen，1.8称之为Metaspace元空间）

2. HotSpot虚拟机 1.6、1.7、1.8运行时数据区的主要差异？

   * 1.7把运行时常量池从永久代PermGen移到堆内存中，为什么？
   * 1.8移除永久代，增加元空间（Metaspace本地内存），为什么？

