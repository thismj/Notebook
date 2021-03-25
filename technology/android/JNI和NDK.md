# JNI和NDK

## JNI跟NDK的关系

`JNI`（`Java Native Interface`）是 `Java` 语言调用` Native`语言的一种特性，它定义了 `Java` 跟 `Native` 代码（C/C++）进行交互的方式

`NDK`（`Native Development Kit`） 是 `Android` 中实现 `JNI` 的开发工具包，用于快速开发 `C/C++` 动态库（`.so`），并通过构建系统 `Grade`将动态库（`.so`）打包进应用 `APK`。`NDK`提供了交叉编译器，用于生成特定的 `CPU` 平台动态库

##[安装及配置 NDK 和 CMake](https://developer.android.com/studio/projects/install-ndk?hl=zh-cn)

## ndk-build方式



## cmake方式

### hello-jni

