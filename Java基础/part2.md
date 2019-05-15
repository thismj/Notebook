# ==、equals() 和 hashCode()

1. Java 中 ==、equals() 和 hashCode() 的作用以及区别？

2. 利用 Objects 类来比较 equals()  以及重写 hashCode() 方法，规避空指针

3. 重写 equals()  方法的原则?
   * 自反性（Reflexive）
   * 对称性（Symmetric）
   * 传递性（Transitive）
   * 一致性（Consistent）
   * 对任何非空对象 x， `x.equals(null)` 必定为 false

4. 重写 hashCode() 方法的原则？
   * 在程序执行期间，只要 equals() 方法的比较操作用到的信息没有被修改，那么对这同一个对象调用多次，hashCode() 方法必须始终如一地返回同一个整数
   * 如果两个对象通过 equals() 方法比较得到的结果是相等的，那么对这两个对象进行 hashCode() 得到的值应该相同
   * 两个不同的对象，hashCode() 的结果可能是相同的，这就是哈希表中的冲突。为了保证哈希表的效率，哈希算法应尽可能的避免冲突
5. 为什么重写 equals() 方法后一定也要重写 hashCode() 方法？

