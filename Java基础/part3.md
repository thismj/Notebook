# String、StringBuffer和StringBuilder

1. String.intern() 方法的作用？

   如果常量池中存在当前字符串, 就会直接返回当前字符串. 如果常量池中没有此字符串, 会将此字符串放入常量池中后, 再返回。在HotSpot虚拟机里面，Java 1.6以及之前的版本，是直接把字符串添加到常量池，1.7版本因为字符串常量池移到了堆内存，所以是把堆内存中该字符串的引用保存在常量池。

2. String为什么不可变？为什么String类被设计成不可变的？

   * final修饰、内部是一个char数组的私有变量、完美封装
   * 安全性（线程安全、防止接口数据篡改）
   * 字符串常量池，节省空间
   * 因为不可变，可以缓存hashCode，适合做Map类集合的Key

3. String、StringBuffer 和 StringBuilder 区别？

   * String是不可变的，StringBuffer 和 StringBuilder是可变的，都是final修饰，内部都是char[]数组，但是StringBuffer 和 StringBuilder实现Appendable来实现可变。
   * StringBuffer是线程安全的，StringBuilder是线程不安全的，不考虑多线程应该用StringBuilder，避免没必要的线程同步消耗。
   * 代码里面如果循环地用 "+" 拼接字符串应该改用 StringBuilder或StringBuffer；简单的"a"+"b"+"c"这种在编译期就能确定的字符串用"+"直接拼接效率最好；假设a是一个字符串变量，那a+"b"这样的代码，通过反汇编class文件发现等同于使用StringBuilder。