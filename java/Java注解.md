# Java注解

Java注解（也被称为元数据），为我们再代码中添加信息提供了一种形式化的方法。

**内置标准注解**：
*  @Override （1.5）
<br/>表示当前方法将覆盖父类的方法，如果方法名称或者签名对不上，则编译器会提示错误信息

* @Deprecated（1.5）
<br/>标记当前元素已过时，编译器会给出警告
* @SuppressWarnings（1.5）
<br/>关闭编译器警告信息，通过字符串来指定需要关闭的编译警告类型，支持的编译警告类型取决于使用的编译器或IDE

* @SafeVarargs （1.7）
<br/>表明该方法内部不会执行类型不安全的操作，可注解于构造函数或者 static、final 修饰的方法，让编译器不提示 "unchecked" 警告

* @FunctionalInterface （1.8）
<br/>表明这是个函数式接口（可转换为 lambda 表达式），如果该接口里面不止一个抽象方法，则编译器会提示错误信息

**内置元注解（负责注解其它的注解）**

* @Target：指定该注解可用的作用域，ElementType 枚举类型包括：

| ElementType | 作用域 |
|---|---|
|TYPE|类、接口、枚举|
|FIELD|类字段（包括枚举常量）|
|METHOD|方法|
|PARAMETER|参数|
|CONSTRUCTOR|构造函数|
|LOCAL_VARIABLE|局部变量|
|ANNOTATION_TYPE|注解类型|
|PACKAGE|包|
|TYPE_PARAMETER|泛型参数（1.8）|
|TYPE_USE|类型名称（1.8）|

* @Retention：指定注解保留的阶段，RetentionPolicy 枚举类型包括：

| RetentionPolicy | 阶段 |
|---|---|
|SOURCE|仅保留在源码阶段，会被编译器丢弃|
|CLASS|编译时保留在 class 文件中，VM运行时丢弃|
|SOURCE|保留到VM运行时，可以通过反射获取信息|

* @Documented：指定生成 JavaDoc 时包含该注解

* @Inherited：指定该注解作用在 class 上时，子类可以继承父类的注解

* @Native：表示定义常量值的字段可以被 native 代码引用（1.8）

* @Repeatable：指定一个容器注解，使该注解可以在一个元素上重复使用（1.8）：

  ```java
   @Repeatable(Authorities.class)
    public @interface Authority {
        String role();
    }

    public @interface Authorities {
        Authority[] value();
    }

    public class RepeatAnnotationUseNewVersion {
        @Authority(role="Admin")
        @Authority(role="Manager")
        public void doSomeThing(){ }
    }
   
  ```
  

**注解处理器**

apt（Annotation processing tool）是 javac 的一个工具





