# Kotlin

Kotlin 名字来源于俄罗斯芬兰湾中一个岛屿的名字，全称是 Kotlin Island，是英语「科特林岛」之意。

## 变量和属性

Java 中的一些关于变量的概念定义：

| 概念                           | 描述                                             |
| ------------------------------ | ------------------------------------------------ |
| *Field*（*字段*）              | 类的成员变量，包括非静态的实例变量、静态的类变量 |
| *Local Variable*（*局部变量*） | 方法体或者静态代码块内定义的变量                 |
| *Parameter*（*参数*）          | 方法调用声明的参数                               |

什么是属性？属性的概念比较抽象，它是一个用户可以改变的对象特征，其本质上而言是可以从对象外部公开访问的Field，Kotlin的属性提供了这样一种一致性，即对象的属性只能通过访问器访问。

Kotlin 用关键字 var 定义可变属性，可以重新赋值；val 定义不可变属性，只能赋值一次。属性可以声明在代码文件顶层，作为顶层属性。

声明一个属性的完整写法：

```kotlin
var <propertyName>[: <PropertyType>] [= <property_initializer>]
    [<getter>]
    [<setter>]
```

为什么会有 *幕后字段* ？首先，在 Kotlin 中得把属性跟 Java 中字段的含义区分开，不管是在类内部还是外部，属性都是通过 getter、setter 来访问的，它不是字段，举个例子：

在 Kotlin 中定义一个属性 name，此时自定义实现它的 getter、setter 访问器，并且不使用 field 字段，此时查看字节码，是没有生成 name 这个字段的，而只有 getName、setName 两个方法而已，从这儿可以看出，虽然有 name 这个属性，但是并没有 name 这个字段。但是不能说它们没有关系，只要 getter、setter 至少有一个使用了默认实现，或者在自定义的 getter、setter 中使用了 field 这个变量，此时就会生成一个 name 属性对应的 name 字段，而此字段则是 kotlin 中所谓的 *幕后字段*，并且幕后字段只能在 getter、setter 里面使用。如果 *幕后字段*  存在，并且没有指定为延迟初始化或延迟加载，也不是抽象的，则必须具有初始化器。

还有一个 *幕后属性* 的概念，官方文档举例的使用场景是，如果类中需要一个属性，对外表现为 val 只读，对内表现为 var 可读写，则可以这么写：

```kotlin
private var _table: Map<String, Int>? = null
public val table: Map<String, Int>
    get() {
        if (_table == null) {
            _table = HashMap() // 类型参数已推断出
        }
        return _table ?: throw AssertionError("Set to null by another thread")
    }
```

如果属性被声明为 private 的话，则默认不会生成 getter、setter，所以使用 *_table*  属性的时候不会产生函数调用开销。 结合 *幕后字段* 的理解，可以认为上述代码只会生成一个名字为 *_table*  的字段，以及一个 *getTable()*  的方法。

疑问，如果仅仅只是要属性对外只读对内可读写的话，下面的方法不是也可以吗：

```kotlin
var table: Map<String, Int>? = null
private set
```

## 