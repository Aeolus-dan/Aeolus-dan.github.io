---
layout: post
title: 测试与开发所依赖的版本不同导致的问题
date: 2017-05-12
categories: blog
tags: [Android,Gradle,Compile]
description: Android androidTestCompile compile gradle

---

# gradle时出现的问题 :
    Error:Conflict with dependency 'com.android.support:support-fragment' in project ':app'. Resolved versions for app (25.3.1) and test app (25.2.0) differ. See http://g.co/androidstudio/app-test-app-conflict for details.
    Error:Conflict with dependency 'com.android.support:support-media-compat' in project ':app'. Resolved versions for app (25.3.1) and test app (25.2.0) differ. See http://g.co/androidstudio/app-test-app-conflict for details.
    Error:Conflict with dependency 'com.android.support:support-compat' in project ':app'. Resolved versions for app (25.3.1) and test app (25.2.0) differ. See http://g.co/androidstudio/app-test-app-conflict for details.
    Error:Conflict with dependency 'com.android.support:support-core-utils' in project ':app'. Resolved versions for app (25.3.1) and test app (25.2.0) differ. See http://g.co/androidstudio/app-test-app-conflict for details.
    Error:Conflict with dependency 'com.android.support:support-core-ui' in project ':app'. Resolved versions for app (25.3.1) and test app (25.2.0) differ. See http://g.co/androidstudio/app-test-app-conflict for details.
# 解决：
  分别对compile和androidTestCompile指定版本.

    Project:
    ext {
        buildToolsVersion = "25.0.2"
        supportLibVersion = "25.2.0"
        supportTestLibVersion = "25.3.1"
        runnerVersion = "0.5"
        rulesVersion = "0.5"
        espressoVersion = "2.2.2"
        archLifecycleVersion = "1.0.0-alpha1"
        archRoomVersion = "1.0.0-alpha1"
    }

    Module:
    androidTestCompile 'com.android.support:support-annotations:' + rootProject.supportTestLibVersion;
    androidTestCompile 'com.android.support:support-v4:' + rootProject.supportLibVersion;
    androidTestCompile 'com.android.support:recyclerview-v7:' + rootProject.supportLibVersion;
    androidTestCompile 'com.android.support:support-fragment:'+rootProject.supportLibVersion;
        

