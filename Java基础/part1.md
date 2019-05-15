# 基本类型

1. java 中共有几种基本数据类型？每种类型各占多少个字节数？

   * 类的成员变量、基本类型数组  **&rarr;** 方法区（Method Area）

   * 局部变量、方法参数 **&rarr;** 虚拟机栈中栈帧的局部变量表（Local Variables）
   * 虚拟机栈中栈帧的操作数栈（Operand Stack）

2. int 与 integer 的区别？自动装箱与自动拆箱？

   * Integer 内部类 IntegerCache 保存了 [-128, 虚拟机参数指定最大值] 的对象缓存 
   * 写个例子用 javap 查看字节码，发现自动装箱与自动拆箱实际上等同于 Integer 的 valueOf() 以及 intValue() 方法 

