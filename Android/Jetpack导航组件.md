# Jetpack导航组件

## 导航原则

* 每个 APP 都有一个固定的首页，即用户在 Luncher 启动 APP 时首先看到的页面以及点击返回键回到 Launcher 时最后看到的页面（不考虑欢迎跟登录注册等页面）。
* 页面导航通过栈的形式来表示，当前页面在栈顶，固定的首页在栈底。
* 在当前 APP 任务里面，标题栏的向上按钮跟系统状态栏的返回键的作用是一样的。
* 当 APP 回到首页的时候，不应该显示标题栏上的向上按钮，只能通过系统的返回键退出当前 APP 。
* Deep Link 进当前 APP 时，需要清除掉 APP 原有的页面栈，并模拟生成手动导航到Deep Link 页面的页面栈，此时点击标题栏的向上按钮可以一步一步回到首页，而点击返回键应该回到产生 Deep Link 的地方。

## 导航组件

主要包含三个部分：

1. 导航图：用 xml 文件集中描述页面相关的所有信息（ res/navigation 资源文件夹）
2. NavHost：页面的空白容器（ 默认实现为 NavHostFragment ）
3. NavController：管理和控制导航页面，通知 NavHost 导航到相应页面

### 使用入门

DrawerLayout + 导航组件实现侧边栏导航：

#### 添加依赖

```bas
dependencies {
  def nav_version = "2.1.0"
  implementation "androidx.navigation:navigation-fragment:$nav_version"
  implementation "androidx.navigation:navigation-ui:$nav_version"
}
```

#### 添加导航图

在 res/navigation 目录添加导航描述文件：













