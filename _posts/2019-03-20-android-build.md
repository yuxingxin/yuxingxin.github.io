---
layout: post
title: Android构建流程分析
tags: build
categories: Android
date: 2019-03-20
---

## APK组成

我们都知道APK其实本质是一个压缩包文件，我们可以通过手动解压或者在Android Studio中的APK分析器我们可以看到，他最终是由这几部分组成：

- Dex文件：.class文件通过d8工具处理后端产物，Android虚拟机可以执行的文件
- Resource文件：资源文件，主要包括layout，drawable，animator等
- resources.arsc：资源索引表
- Assets文件：资源文件，通过AssetManager来加载
- Library：so库文件
- META-INF：APK签名信息
- AndroidManifest.xml：清单文件

## 构建流程

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h5bxs1rdr1j20qe0tomz2.jpg)

上面这张图是现在的官方构建流程图，但是不够详细，下面这个是早期的android构建流程图，看起来更清楚一些，图中绿色标注为其中用到的相应工具，蓝色代表的是中间生成的各类文件类型：

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h5bwj82sjbj20ew0oiab0.jpg)

我们的应用程序通常由这几部分组成

- 源代码文件
- AIDL文件
- 资源文件
- 清单文件

我们构建的过程其实就是将上面几种文件编译成APK解压缩后的文件，其大致的一个框架是：

1. 首先aapt工具会将资源文件进行转化，生成对应资源ID的R文件和二进制资源文件。

1. adil工具会将其中的aidl接口转化成Java的接口

1. 至此，Java Compiler开始进行Java文件向class文件的转化，将R文件，Java源代码，由aidl转化来的Java接口，统一转化成.class文件。

1. 通过dx工具将class文件转化为dex文件，现在通过D8工具完成这一过程转化。

1. 此时我们得到了经过处理后的资源文件和一个dex文件，当然，还会存在一些其它的资源文件，这个时候，就是将其打包成一个类似apk的文件。但还并不是直接可以安装在Android系统上的APK文件。

1. 通过签名工具对其进行签名。

1. 通过Zipalign进行优化，提升运行速度。

更细致的分析我们通过另外一张图可以了解：

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h5bwybxh9fj20rl0u3dk9.jpg)

接下来我们先准备上面需要构建的几类文件，然后结合这张图分析下具体流程：

1. **通过aapt2 compile 打包资源文件，生成编译后的.flat文件（二进制文件）** 

   这一步我们可以通过命令行执行就是：`aapt2 compile -o build/res.zip --dir res`

2. **通过 aapt2 link 将 .flat 和 AndroidManifest 进行连接，转化成不包含 dex 的 apk 和 R.java**

   同样用命令行执行：`aapt2 link build/res.zip -I $ANDROID_HOME/platforms/android-30/android.jar --java build --manifest AndroidManifest.xml -o build/app-debug.apk`

1. **通过 javac 将 Java 文件编译成 .class 文件**

   命令行：`javac -d build -cp $ANDROID_HOME/platforms/android-30/android.jar com/**/**/**/*.java`

2. **通过 d8 将 .class 文件转化成 dex 文件**

   `d8 --output build/ --lib $ANDROID_HOME/platforms/android-30/android.jar build/com/yuxingxin/gradle_test/androidbuild/*.class`

3. **合并 dex ⽂件和资源⽂件**

   `zip -j build/app-debug.apk build/classes.dex`

4. **对 apk 通过 apksigner 进行签名**

   `apksigner sign -ks ~/.android/debug.keystore build/appdebug.apk`

5. **zipalign进行优化，提升运行速度**

   `zipalign -p -f -v 4 appdebug.apk app_debug.apk`

上面过程注意dex文件，因为Android有64K限制，即当 type_ids、method_ids 或者 field_ids 超过 65536（64 * 1024）的时候，需要进行 dex 分包，为了 Dex 的数量尽可能少，我们需要尽量实现「Dex 信息有效率」的提升。

