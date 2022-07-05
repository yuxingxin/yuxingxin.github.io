---
layout: post
title: Android Studio 和Gradle Plugin 3.0 迁移不完全指南
tags: Android Studio
categories: Android
date: 2017-11-03
---
Android Studio 3.0 默认Gradle版本为4.1，如果你需要手动升级版本的话，记得修改gradle/wrapper/gradle-wrapper.properties文件的URL地址：

```
distributionUrl=https\://services.gradle.org/distributions/gradle-4.1-all.zip
```

对应的Gradle插件版本为3.0.0，手动修改的话，需要修改项目级的build.gradle文件：

```
buildscript {
    repositories {
        ...
        // You need to add the following repository to download the
        // new plugin.
        google()
    }

    dependencies {
        classpath 'com.android.tools.build:gradle:3.0.0'
    }
}
```

注意上面记得添加google这个repository，某些官方依赖需要下载

对应我们的构建工具buildToolsVersion版本为26.0.2，对应Module级项目build.gradle文件：

```
android {
    compileSdkVersion 26
    ...
    defaultConfig {
      ...
    }
}
```

## 使用变体感知（variant-aware）依赖管理机制

Android 3.0的插件使用一种新的依赖机制，这种机制能自动的匹配我们项目中依赖库的变体，即app变体debug会自动消费它所依赖的library的debug变体。当然了，我们在给产品定制不同的风味时，它依然能够适用。所以呢为了保证能够准确的匹配这些变体，我们需要为所有的产品风味声明风味维度（flavor dimensions），以及不可能直接匹配的需要我们提供matching fallbacks（PS：不知道怎么翻译了...）

好了，上面说了那么多"废话"，其实只是想说明一点，如果你项目中用了build type 或者product flavor一种或一种以上， 那么你就需要注意了，这里可能需要做相应的适配：

### 添加风味维度的声明

当我们在配置文件中配置产品风味的时候，现在需要声明风味维度，然后在每个产品风味中指定你前面所声明的某一个风味维度，如下：

```
//定义两个风味维度
flavorDimensions "api", "mode"

productFlavors {
    demo {
        //指定风味维度
        dimension "mode"
        ...
    }

    full {
        dimension "mode"
        ...
    }

    minApi24 {
        dimension "api"
        minSDKVersion '24'
        versionNameSuffix "-minApi24"
    }

    minApi23 {
        dimension "api"
        minSDKVersion '23'
        versionNameSuffix "-minApi23"
    }

    minApi21 {
        dimension "api"
        minSDKVersion '21'
        versionNameSuffix "-minApi21"
    }
}
   ```

如上，配置完后，Gradle创建的构建变体数量等于每个风味维度中的风味数量与你配置的构建类型数量的乘积，在 Gradle 为每个构建变体或对应 APK 命名时，属于较高优先级风味维度的产品风味首先显示，之后是较低优先级维度的产品风味，再之后是构建类型。以上面的构建配置为例，Gradle 可以使用以下命名方案创建总共 12 个构建变体：
构建变体：[minApi24, minApi23, minApi21][Demo, Full][Debug, Release]
对应 APK：app-[minApi24, minApi23, minApi21]-[demo, full]-[debug, release].apk
例如构建变体：minApi24DemoDebug，对应 APK：app-minApi24-demo-debug.apk
当然如果有些特定的变体不是你需要的，你也可以过滤：

```
android{
    variantFilter { variant ->
        def names = variant.flavors*.name
        // To check for a certain build type, use variant.buildType.name == "<buildType>"
        if (names.contains("minApi21") && names.contains("demo")) {
        // Gradle ignores any variants that satisfy the conditions above.
        setIgnore(true)
        }
    }
}
```

如果组合多个产品风味，产品风味之间的优先级将由它们所属的风味维度决定。上面所列示的第一个风味维度中的产品风味比第二个维度中的产品风味拥有更高的优先级，以此类推。此外，与属于各个产品风味的源集相比，你为产品风味组合创建的源集拥有更高的优先级。

如果不同源集包含同一文件的不同版本，Gradle 将按以下优先顺序决定使用哪一个文件（左侧源集替换右侧源集的文件和设置）：

构建变体 > 构建类型 > 产品风味 > 主源集 > 库依赖项

如以下优先级顺序：

* src/demoDebug/（构建变体源集）
* src/debug/（构建类型源集）
* src/demo/（产品风味源集）
* src/main/（主源集）

这里说下源集的概念，Android Studio 按逻辑关系将每个模块的源代码和资源分组称为源集，默认情况下，Android Studio 会创建 main/源集和目录，用于存储要在所有构建变体之间共享的一切资源。然而，我们也可以创建新的源集来控制 Gradle 要为特定的构建类型、产品风味（以及使用风味维度时的产品风味组合）和构建变体编译和打包的确切文件。例如，可以在 main/ 源集中定义基本的功能，使用产品风味源集针对不同的客户更改应用的品牌，或者仅针对使用调试构建类型的构建变体包含特殊的权限和日志记录功能等。

* `src/main/`：此源集包括所有构建变体共用的代码和资源。
* `src/<buildType>/`：创建此源集可加入特定构建类型专用的代码和资源。
* `src/<productFlavor>/`：创建此源集可加入特定产品风味专用的代码和资源。
* `src/<productFlavorBuildType>/`：创建此源集可加入特定构建变体专用的代码和资源。

配置完后我们可以通过Android Studio窗口右侧Gradle，导航至YourApplication>Tasks>android下双击sourceSets，在Android Studio底部右下角 Gradle Console处查看项目是如何组织源集的

### 关于构建类型的配置

假设App中配置了一个叫做"jniDebug"的构建类型，但是该App所依赖的库中没有配置，这时候当我们构建"jniDebug"的时候，插件就不知道库该使用什么构建类型，这时候就会给报出下面的错误：

```
Error:Failed to resolve: Could not resolve project :mylibrary.
Required by:project :app
```

这类问题就是由于上面的依赖管理机制变化导致的，我们可以下面的几种情况来分别解决：

#### 你的 Module App 包含了它所依赖的库没有的构建类型

例如，我们的App包含了一个jniDebug的构建类型，但是它所依赖的库中没有这个，而是有debug和release这两个构建类型，这时候我们就可以在Module App的build.gradle文件中使用matchingFallbacks 来指定可以替换的匹配项，如下：

```
android {
    buildTypes {
        debug {}
        release {}
        jniDebug {
            // Specifies a sorted list of fallback build types that the
            // plugin should try to use when a dependency does not include a
            // "jniDebug" build type. You may specify as many fallbacks as you
            // like, and the plugin selects the first build type that's
            // available in the dependency.
            matchingFallbacks = ['debug', 'release']
        }
    }
}
```

值得一提的是插件会选择matchingFallbacks列表中第一个可用的构建类型来替换匹配项。

> 注意当依赖的库中包含了Module App没有的构建类型，则不会出现上述问题。

#### 对于一个给定的存在于App和它所依赖的库中的风味维度，我们的主Module App包含了库中没有的风味

例如，主Module App和库中都包含了一个mode的风味维度，我们的App中指定mode维度的是free和paid风味，而库中指定mode维度的是demo和paid风味，这时候我们就可以用`matchingFallbacks 来为App中的free指定可以替换的匹配项。如下：

```
android {
    defaultConfig{
    // Do not configure matchingFallbacks in the defaultConfig block.
    // Instead, you must specify fallbacks for a given product flavor in the
    // productFlavors block, as shown below.
    }
    flavorDimensions 'mode'
    productFlavors {
        paid {
            dimension 'mode'
            // Because the dependency already includes a "paid" flavor in its
            // "mode" dimension, you don't need to provide a list of fallbacks
            // for the "paid" flavor.
        }
        free {
            dimension 'mode'
            // Specifies a sorted list of fallback flavors that the plugin
            // should try to use when a dependency's matching dimension does
            // not include a "free" flavor. You may specify as many
            // fallbacks as you like, and the plugin selects the first flavor
            // that's available in the dependency's "mode" dimension.
            matchingFallbacks = ['demo', 'trial']
        }
    }
}
```

值得注意的是，上述情况中，如果说库中包含了一个主Module App没有的产品风味，则不会出现上述问题。

#### 库中包含了一个主Module App没有的风味维度

例如，库中声明了一个minApi的风味维度，但是你的App中只有mode维度，因此当你要构建freeDebug这个变种版本的App时，插件就不知道你是想用minApi23Debug还是用minApi25Debug变种版本的库，这时候我们可以在主Module App中的defaultConfig代码块通过配置missingDimensionStrategy来让插件从丢失的维度中指定默认的风味，当然你也可以在productFlavors代码块中覆盖先前的选择，因此每一个风味都可以为丢失的维度指定一个不同的匹配策略。

```
android {
    defaultConfig{
        // Specifies a sorted list of flavors that the plugin should try to use from
        // a given dimension. The following tells the plugin that, when encountering
        // a dependency that includes a "minApi" dimension, it should select the
        // "minApi23" flavor. You can include additional flavor names to provide a
        // sorted list of fallbacks for the dimension.
        missingDimensionStrategy 'minApi', 'minApi23', 'minApi25'
        // You should specify a missingDimensionStrategy property for each
        // dimension that exists in a local dependency but not in your app.
        missingDimensionStrategy 'abi', 'x86', 'arm64'
    }
    flavorDimensions 'mode'
    productFlavors {
        free {
            dimension 'mode'
            // You can override the default selection at the product flavor
            // level by configuring another missingDimensionStrategy property
            // for the "minApi" dimension.
            missingDimensionStrategy 'minApi', 'minApi25', 'minApi23'
        }
        paid {}
    }
}
```

> 值得注意的是，当你的主Module App中包含了一个库中依赖项没有的风味维度时，则不会出现上述问题。例如，当库中依赖项不包含abi这个维度时，freeX86Debug版本将会使用freeDebug版本的依赖。

## 使用新的依赖配置

先来看下以前的配置：

```
android {...}
...
dependencies {
    // The 'compile' configuration tells Gradle to add the dependency to the
    // compilation classpath and include it in the final package.

    // Dependency on the "mylibrary" module from this project
    compile project(":mylibrary")

    // Remote binary dependency
    compile 'com.android.support:appcompat-v7:26.1.0'

    // Local binary dependency
    compile fileTree(dir: 'libs', include: ['*.jar'])
}
```

上面也是我们常见的三种依赖项声明：

1. 模块依赖项
compile project(':mylibrary') 行声明了一个名为“mylibrary”的本地 Android 库模块作为依赖项，并要求构建系统在构建应用时编译并包含该本地模块。
2. 远程二进制依赖项
compile 'com.android.support:appcompat-v7:26.1.0' 行会通过指定其 JCenter 坐标，针对 Android 支持库的 26.1.0 版本声明一个依赖项。
3. 本地二进制依赖项
compile fileTree(dir: 'libs', include: ['*.jar']) 行告诉构建系统在编译类路径和最终的应用软件包中包含 app/libs/ 目录内的任何 JAR 文件。

先来看下以前配置的关键字：

* compile
指定编译时依赖项。Gradle 将此配置的依赖项添加到类路径和应用的 APK。这是默认配置。

* apk
指定 Gradle 需要将其与应用的 APK 一起打包的仅运行时依赖项。我们可以将此配置与 JAR 二进制依赖项一起使用，而不能与其他库模块依赖项或 AAR 二进制依赖项一起使用。

* provided
指定 Gradle 不与应用的 APK 一起打包的编译时依赖项。如果运行时无需此依赖项，这将有助于缩减 APK 的大小。我们可以将此配置与 JAR 二进制依赖项一起使用，而不能与其他库模块依赖项或 AAR 二进制依赖项一起使用。

* 使用构建变体或者测试源集的名称配置关键字
这种方式可以为特定的构建变体或者测试源集配置依赖项，如：

```
dependencies {
    ...
    // Adds specific library module dependencies as compile time dependencies
    // to the fullRelease and fullDebug build variants.
    fullReleaseCompile project(path: ':library', configuration: 'release')
    fullDebugCompile project(path: ':library', configuration: 'debug')

    // Adds a compile time dependency for local tests.
    testCompile 'junit:junit:4.12'

    // Adds a compile time dependency for the test APK.
    androidTestCompile 'com.android.support.test.espresso:espresso-core:2.2.2'
}
```

我们再来看下新的配置项：

  * implementation

     原有compile已经废弃掉，新增了implementation，用它来配置模块时，它是用来告诉Gradle该Module不会在编译时暴露其依赖给其他Module，而仅仅是在运行时才会暴露出来，即对其他Module可用。因此它也常常用在该模块不需要有别的模块依赖时声明使用，例如我们的App Module或者test Module。这样做的好处是减少了我们构建的时间。

  * api

     同原有compile，它和上面的区别就是它在编译器和运行期都会暴露它所配置的模块，因此也常常用在								library module中。这是因为一旦我们api配置的这个Module有变化，Gradle会在编译期重新编译那个依赖项的所有依赖。所以，如果我们项目中有大量的api配置项依赖，那么无形中就增加了构建的时间，除非你想暴露这个模块的API给其他Module使用，否则，我们应尽可能使用implementation来代替。

  * compileOnly

     同上面的provided，只在编译时用，不会打包到我们的APK中

  * runtimeOnly

     同上面的apk

## 一些API的变化

尤其注意的是我们重命名打包的APK文件，以及输出路径。
变化前：

```
applicationVariants.all { variant ->
    variant.outputs.each { output ->
        def outputFile = output.outputFile
        if (outputFile != null && outputFile.name.endsWith('.apk')) {
            if (variant.buildType.name == 'lotteryTest') {
                def fileName = "myApp_v${defaultConfig.versionName}_${releaseTime()}.apk"
                output.outputFile = new File(outputFile.parent, fileName)
            }
        }
    }
}
```

变化后：

```
applicationVariants.all { variant ->
    variant.outputs.all { output ->
        def outputFile = output.outputFile
        if (outputFile != null && outputFile.name.endsWith('.apk')) {
            if (variant.buildType.name == 'lotteryTest') {
                def fileName = "myApp_v${defaultConfig.versionName}_${releaseTime()}.apk"
                outputFileName = new File(fileName)
            }
        }
    }
}
```

即我们需要修改each() 和 outputFile() 方法为 all() 和 outputFileName


## 默认启用AAPT2
  在迁移的过程中，如果发现由于aapt2导致的异常，可以在gradle.properties中加入：

  ```
  android.enableAapt2=false
  ```

## 支持Java8新特性

### Gradle带来全新的Java8支持方案desugar

该方案启用十分简单，只需要配置下面代码：

```
compileOptions {
    sourceCompatibility JavaVersion.VERSION_1_8
    targetCompatibility JavaVersion.VERSION_1_8
}
```

如果你不想使用，也可以禁用，可以在gradle.properties中加入：

```
android.enableDesugar=false
```

记得删除上面的兼容Java8代码。

### 移除Jack工具链，不再支持

```
android {
...
defaultConfig {
    ...
    // Remove this block.
    jackOptions {
        enabled true
        ...
    }
}
// Keep the following configuration in order to target Java 8.
compileOptions {
    sourceCompatibility JavaVersion.VERSION_1_8
    targetCompatibility JavaVersion.VERSION_1_8
}
}
```

### 移除Retrolambda插件

项目级build.gradle 文件：

```
buildscript {
...
dependencies {
    // Remove the following dependency.
    classpath 'me.tatarka:gradle-retrolambda:<version_number>'
}
}
```

Module级build.gradle文件：

```
// Remove the following plugin.
apply plugin: 'me.tatarka.retrolambda'
...
// Remove this block after migrating useful configurations.
retrolambda {
...
// If you have arguments for the Java VM you want to keep,
// move them to your project's gradle.properties file.
jvmArgs '-Xmx2048m'
}
```

### 目前兼容支持的功能特性有：

* Lambda expressions
* Method References
* Type Annotations
* Default and static interface methods
* Repeating annotations

## 参考

* https://developer.android.com/studio/write/java8-support.html#migrate
* https://android-developers.googleblog.com/2017/10/android-studio-30.html
* https://developer.android.com/studio/build/gradle-plugin-3-0-0.html#known_issues
* https://developer.android.com/studio/build/gradle-plugin-3-0-0-migration.html