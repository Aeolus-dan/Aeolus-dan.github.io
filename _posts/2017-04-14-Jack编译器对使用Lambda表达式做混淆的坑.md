---
layout: post
title: Jack 编译器对混淆lambda代码的影响
date: 2017-04-14
categories: blog
tags: [Android,Jack]
description: Android Jack 编译器

---

# 起因 :
Jack 是 Java Android Compiler Kit 的缩写，它可以将 Java 代码直接编译为 Dalvik 字节码，并负责 Minification, Obfuscation, Repackaging, Multidexing, Incremental compilation。它试图取代 javac/dx/proguard/jarjar/multidex 库等工具。

<br><br>
Android studio从2.2开始支持Java8的部分新特性。也就意味着终于可以用上官方的lambda, 但是问题来了...



在Android studio 中使用Java8, 可以通过如下配置
<pre><code>
 android{
       ...
        compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
        defaultConfig {
             ...
         jackOptions {
             enabled true
          }
      }
        ...
 }
</code></pre>

在一切配置完成后就可以愉快的使用lambda来缩短代码量。

But。。。。。。。

当项目完成时对代码进行混淆时出现了

    11-15 01:46:26.818: W/System.err(21810): java.lang.RuntimeException: Missing type parameter.
    11-15 01:46:26.828: W/System.err(21810):    at da.<init>(Unknown Source)
    11-15 01:46:26.828: W/System.err(21810):    at gc.<init>(Unknown Source)
    11-15 01:46:26.828: W/System.err(21810):    at fx.f(Unknown Source)
    11-15 01:46:26.828: W/System.err(21810):    at com.yourshows.activity.UnwatchedActivity.onResume(Unknown Source)


# 解决：
1. 看到这个错误是一脸懵逼啊。。<br>没办法决定先查看/build/outputs/mapping/mapping.txt文件下的映射关系
    com.google.gson.reflect.TypeToken -> da:
经过一番折腾在网上找到了相关解决方法：<br>
  在混淆文件中加入对GSON的混淆<br><br>
<pre><code>
    # Gson uses generic type information stored in a class file when working with fields. Proguard<br>
    # removes such information by default, so configure it to keep all of it.<br>
    -keepattributes Signature<br>

    # Gson specific classes<br>
    -keep class sun.misc.Unsafe { *; }<br>
    #-keep class com.google.gson.stream.** { *; }<br>

    # Application classes that will be serialized/deserialized over Gson<br>
    -keep class com.google.gson.examples.android.model.** { *; }<br>
    -keep class com.google.gson.** {*;}<br>
    -keep class org.json.** {*;}<br>
    -keepattributes *Annotation*<br>
    -keep class * implements com.google.gson.TypeAdapterFactory<br>
    -keep class * implements com.google.gson.JsonSerializer<br>
    -keep class * implements com.google.gson.JsonDeserializer<br>
</code></pre>

满心欣喜的以为解决了问题, 可是这只是自己的期盼。 在看到程序崩溃的时候，心情低落到了低谷.

2.灵光一闪,想起之前在官方未支持lambda时,一直使用retrolambda,混淆时没有出现这些问题.这时就想难道是新编译器的问题. 接着找应该保留的类，在这期间，保留了java.lang.invoke包下的所有类，但是都没有解决。不得已只能使用retrolambda了, <br>
    然后这个问题就被规避了.<br><br>
    **记得将enable 设为false。 不然会与retrolambda 冲突. 针对于支持Jack编译器的studio**

3. 使用Jack时还可能在使用`shrinkResources true`时出现:<br>
<pre><code> Error:A problem was found with the configuration of task ':app:packageBaiduRelease'.
File 'F:\myjob\github\Android-architecture-todo-mvp-rxjava\MVP\app\build\intermediates\res\resources-baidu-release-stripped.ap_' specified for property 'resourceFile' does not exist.</code></pre>



