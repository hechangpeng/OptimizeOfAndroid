---
title: 性能优化的一点实践
date: 2017-10-09 14:06:10
tags:
categories: Android
---
<img src="http://owiq5fnuk.bkt.clouddn.com/9.jpg"/>
最近做的项目在测试中发现了一个闪退的异常，于是跟踪log发现是内存溢出导致的crash，于是着手解决该问题，在解决这个BUG的过程中发现了很多以前没有发现的问题。<!--more-->小组leader总是叫我们要做好monkey自动化测试、性能优化检测的一些工作，自己没有很重视这一块，现在就特别后悔啊。感觉性能优化推动起来有点像秦国的商鞅变法，简直是功在当代，利在千秋的伟大事业啊。其实，很多人包括我也是当初不重视这一块，很多时候我们会觉得这是在打脸的事情，因为性能不好往往就是代码写的不好或者某些地方缺乏思考引起的，有时候明明意识到问题却把责任归咎于是没时间啊、这些都是遗留问题啊、改起来牵扯比较多、还要做新功能呢等等。
好吧，打脸了。但是，善于自我批评才能有进步哦。
进入正题吧。

#### 一、Android Studio的代码分析工具

Analyze —> inspect code

1.Context持有的宿主引用引起的内存泄漏。<br/>
2.Handler非静态内部类引起的内存泄漏。<br/>
3.布局嵌套层次太深引起的绘制慢UI不流畅。<br/>
4.增强for循环替代循环，条件判断语句、return语句简化。<br/>
5.代码冗余问题。<br/>
6.函数职责不单一，代码太长太丑问题。<br/>
7.静态实例持有context的外部引用。<br/>
8.数据库操作游标未关闭、文件操作输入输出流未关闭引起的内存泄漏。<br/>
9.Bitmap回收与图片压缩。<br/>
10.缓存导致的资源没有释放的问题。<br/>

#### 二、Memory Analysis Tools 分析dump文件

Android Monitors —> dump java heap —> xxx.dump

大致的思路是将得到的dump文件通过 hprof-conv (位于 sdk/tools)转化为hprof文件，再用MAT打开该hprof文件。

#### 三、LeakCanary

在 build.gradle 中加入引用，不同的编译使用不同的引用：

 dependencies {
   debugCompile 'com.squareup.leakcanary:leakcanary-android:1.3'
   releaseCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.3'
 }

在Application中加入：
  
  LeakCanary.install(this);
  
注：我用这种方法有时候检测到了内存泄漏但不显示内存泄漏栈信息，但在有的手机上又是ok的。。。
#### 四、使用Android StrictMode检测代码

首先开启严苛模式：

        StrictMode.ThreadPolicy.Builder builder = new StrictMode.ThreadPolicy.Builder();
        builder.detectDiskReads()
                .detectDiskWrites()
                .detectNetwork();
        if (Build.VERSION.SDK_INT >= 23) {
            builder.detectResourceMismatches();
        }
        builder.detectCustomSlowCalls()
                .penaltyLog()
                .penaltyDeath();
        StrictMode.setThreadPolicy(builder.build());
        StrictMode.setVmPolicy(new StrictMode.VmPolicy.Builder()
                .detectActivityLeaks()
                .detectLeakedClosableObjects()
                .detectLeakedRegistrationObjects()
                .detectLeakedSqlLiteObjects()
                .penaltyLog()
                .penaltyDeath()
                .build());

然后adb命令开启日志收集：adb logcat > log.txt ，当检测到不合理的代码时（如主线程处理了耗时的数据库、文件、网络任务，游标未关闭、Activity Two instance等），程序会闪退，根据闪退log信息来定位修改，有时候若无法定位到是哪里引起的内存泄漏，则要根据上面的第一种方法列出的各项好好审查自己的代码了。
这个方法真的挺不错的。

