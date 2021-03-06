# Android架构

[MVC，MVP 和 MVVM 的图示](https://www.ruanyifeng.com/blog/2015/02/mvcmvp_mvvm.html)

## MVC

![](https://www.ruanyifeng.com/blogimg/asset/2015/bg2015020105.png)



## MVP

![](https://www.ruanyifeng.com/blogimg/asset/2015/bg2015020109.png)

### 优势

* 面向接口编程，Presenter 层作为 Model 层跟 View 层的交互中介，实现了视图与数据的解耦，便于高效并行开发、单元测试等。
* MVP各层职责清晰，代码具有更好的可阅读性和可维护性，在接口不变的前提下修改某层的代码不会影响到其他层。

### 缺点

* 接口粒度不好控制，粒度太小，会导致代码量大大增加，粒度太大，会导致 Presenter 层跟 MVC 的 Controller 层一样臃肿
* Presenter 层对生命周期无感知，需要手动添加相应的生命周期方法，比较麻烦
* Presenter 层跟 View 层存在耦合，View 层修改时可能会导致 Presenter 层大量变动

## MVVM

![](https://www.ruanyifeng.com/blogimg/asset/2015/bg2015020110.png)

核心：依赖于生命周期感知组件 Lifecycle 的 ViewModel 和 LiveData。

### 优势

* 当设备配置发生改变时，ViewModel 会保存数据，所以无需从数据库或网络重新加载数据
* LiveData 具备生命周期感知能力，无需在更新数据时考虑生命周期的问题。



## 模块化、组件化

### 模块如何划分



### 代码以及资源隔离



### 组件间通信



### 组件单独编译及运行









