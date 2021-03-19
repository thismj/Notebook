# Gradle

## Groovy基础



## Android Gradle 配置

Android Gradle 插件配置官方参考文档：[Android Gradle plugin API reference 4.1](https://developer.android.com/reference/tools/gradle-api/4.1/classes)

### 配置 Build variants

Build Variants 由 `buildTypes` 和 `productFlavors` 组合而成，`buildTypes` 代表构建类型、`productFlavors` 代表产品风格，一般来说 `buildTypes` 用于区分软件周期的不同阶段，例如开发时用 `debug`，正式发布时用 `release`，本质上是同一个产品，功能都是一样的；`productFlavors` 用来区分不同的产品类型，包名、icon、功能也许都会有差异。

插件会默认配置 `debug` 跟 `release` 两种 buildTypes，其中 `debug` 的 `debuggable` 属性设置为 `true`

```groovy
android {
    ...
    buildTypes {
        //如果使用默认的设置，不需要显示配置出来
        debug {}
        release {
            //applicationId后缀
            applicationIdSuffix "."
            //是否可以debug
            debuggable false
            //是否能debug native代码
            jniDebuggable false
            //是否启用代码缩减（插件版本>=3.4.0时，使用R8编译器处理代码缩减、混淆等工作）
            minifyEnabled true
            //移除未使用的资源，需要minifyEnabled配合
            shrinkResources true
            //指定混淆配置文件
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
    ...
}
```

配置 `productFlavors`，支持跟 `defaultConfig` 相同的属性配置：

```groovy
android {
    ...
    //指定维度种类
    flavorDimensions "version"
    productFlavors {
        demo {
            //flavor必须指定所属维度，如果只有一个维度则默认为该维度
            dimension "version"
            applicationIdSuffix ".demo"
            versionNameSuffix "-demo"
        }
        full {
            dimension "version"
            applicationIdSuffix ".full"
            versionNameSuffix "-full"
        }
    }
}
```

组合生成所有 Build Variants，按照 `<product-flavor><Build-Type>` 命名：

```bash
demoDebug
demoRelease
fullDebug
fullRelease
```



### 配置构建依赖

| 配置                | 行为                                                         | ---  |
| ------------------- | ------------------------------------------------------------ | ---- |
| implementation      | 参与编译并打包进apk，但是依赖关系不传递，使用 implementation 替代 api，可以减少重新编译的模块数，缩短构建时间 |      |
| api                 | 参与编译并打包进apk，传递依赖关系                            |      |
| compileOnly         | 仅仅参与编译，不打包进apk                                    |      |
| runtimeOnly         | 仅仅打包进apk，不参与编译                                    |      |
| annotationProcessor | 注释处理器                                                   |      |
| lintChecks          |                                                              |      |
| lintPublish         |                                                              |      |

上面的配置选项会作用于所有的 Build Variants，如果想单独对某个 `buildTypes` 和 `productFlavors` 做依赖配置，则直接配置如下：

```groovy
dependencies {
    debugImplementation 'com.google.firebase:firebase-ads:9.8.0'
    demoImplementation 'com.google.firebase:firebase-ads:9.8.0'
}
```

如果想对某个特定的通过  `buildTypes` 和 `productFlavors` 组合的  Build Variants 做依赖配置，需要在 `configurations` 中先初始化配置名字：

```groovy
configurations {
    // Initializes a placeholder for the freeDebugRuntimeOnly dependency
    // configuration.
    demoDebugRuntimeOnly {}
}

dependencies {
    demoDebugRuntimeOnly fileTree(dir: 'libs', include: ['*.jar'])
}
```

**annotationProcessor**

如果依赖（`compileOnly` 等）具有注解处理器的模块，插件会报错，必须改为显示使用 `annotationProcessor` 来添加注解处理器，当然也可以通过如下配置来停用这个错误检查，这样注解处理器也就不起作用了：

```groovy
android {
    ...
    defaultConfig {
        ...
        javaCompileOptions {
            annotationProcessorOptions {
                includeCompileClasspath false
            }
        }
    }
}
```

