---
title: Android Studio使用小全
date: 2016-06-30 13:04:31
tags:
- android studio
- gradle
---
* [面板](#panel)
* [快捷键](#keymap)
* [Gradle Build Tool](#gradle-build-tool)
    * [gradle](#gradle)
    * [android-plugin](#android-plugin)
    * [gradlew](#gradlew)
    * [编译加速](#make-speed)
* [插件](#plugin)

<h2 id="panel">面板</h2>
![main panel](/uploads/main_panel.png)  

1. 工具栏  
  主要按钮：run，debug, attach进程调试, 设置, gradle sync, svn update/commit/compare/history, project structure  

2. 导航栏  
方便文件定位，也可在设置中移除  
3. 编辑窗口  
  <table>
  <tr>
  <td>Ctrl+Shift+F12</td><td>定位并放大编辑区</td>
  </tr>
  <tr>
  <td>Ctrl+左击文件tab</td><td>在文件系统打开</td>
  </tr>
  <tr>
  <td>Shift+左击文件tab</td><td>关闭该文件</td>
  </tr>
  </table>
4. 工具窗口  
  常用的工具栏会对应一个下标，Alt+数字下标就能快捷切换窗口  
  0\. message: gradle sync信息  
  1\. project: 项目的结构视图，按功能划分了多种结构，每个都点一下感受一下  
  2\. favorite: `收藏`用来收藏常用类，`书签`可保存某一行的定位，`断点`管理断电  
  3\. find: 查找结果  
  4\. run: 查看安装启动应用的命令  
  5\. debug: 分debugger和console两个面板  
  调试时，Alt+点击表达式可计算结果  
  有个蓝色设置按钮，勾选显示返回结果  
  右击断点可设置用log代替断点，log在console输出  
  6\. monitor: 应用运行的状态，有几个小按钮点点看是干嘛的  
  7\. structure: 类结构  
  9\. version control: 版本控制

5. 状态栏  
左下角按钮可用于显示/隐藏工具窗口标签

<h2 id="keymap">快捷键</h2>
更多快捷键查看[官方文档](https://developer.android.com/studio/intro/keyboard-shortcuts.html)(如果快捷键是eclipse的模板，有些会不适用)  

| Keys     | Usage     |
| :------------- | :------------- |
|Ctrl+E|最近打开过的文件|
|Ctrl+Shift+E|最近编辑过的文件|
|Ctrl+Shift+F12|扩大/缩小代码区|
|Ctrl+Shift+Number|设置标签|
|Ctrl+Number|定位标签|
|Alt+Shift+F|添加到Favorities|
|Alt+Number|打开指定Panel|
|Alt+Insert|生成构造器、Override等常用方法|
|Ctrl+Alt+Shift+N|查找属性/方法|

## Gradle Build Tool
#### gradle
Android Studio集成的编译工具  
手动下载链接：http://services.gradle.org/distributions  
镜像：http://android-mirror.bugly.qq.com:8080/gradle/  
手册：https://docs.gradle.org/current/dsl/  
用户引导：http://tools.android.com/tech-docs/new-build-system/user-guide

#### android-plugin  
gradle的build script，用groovy动态语言编写编译代码  
手动下载链接：https://jcenter.bintray.com/com/android/tools/build/gradle/  
手册：http://google.github.io/android-gradle-dsl/current/index.html

#### gradlew  
是对gradle的封装，通过设置gradle-wrapper.properties指定gradle版本  
如果没有设置GRADLE_USER_HOME环境变量，默认下载的gradle包在C:/Users/xx/.gradle/wrapper/dists/xxxxx下面  
如果下载慢可手动下载zip包然后拷贝到这个目录下  

```
gradlew -h  
gradlew clean  //清除所有module编译结果
gradlew --gui  //可视化命令窗口
gradlew --stop  //停止gradle daemon进程，清理内存  
gradlew --info --profile  //增加log输出，dump编译时间
gradlew assembleDebug  //编译debug版本
gradlew installDebug  //编译并安装debug版本
```

<h4 id="make-speed">编译加速</h2>
1. 更新Android Studio
2. 更新gradle(在gradle-wrapper.properties里面配置)
3. 更新android-plugin
4. 配置gradle.properties
```
android.useDeprecatedNdk=true
org.gradle.jvmargs=-Xmx2560M -XX:MaxPermSize=512m -XX:+HeapDumpOnOutOfMemoryError -Dfile.encoding=UTF-8
org.gradle.daemon=true
org.gradle.configureondemand=true
org.gradle.parallel=true
```
5. 配置app的build.gradle，添加
```
productFlavors {
    // Define separate dev and prod product flavors.
    dev {
        // dev utilizes minSDKVersion = 21 to allow the Android gradle plugin
        // to pre-dex each module and produce an APK that can be tested on
        // Android Lollipop without time consuming dex merging processes.
        minSdkVersion 21
    }
    prod {
        // The actual minSdkVersion for the application.
        minSdkVersion 14
    }
}
```
6. 使用5.0及以上真机或模拟器，build variable选择devDebug

<h2 id="plugin">插件</h2>
在Settings->Plugins输入插件标题，搜索安装即可  
https://mp.weixin.qq.com/s?__biz=MzI3MDE0NzYwNA==&mid=2651433634&idx=1&sn=e5f65d8a0a2b85f7c22d8ccd4cf96a39&scene=1&srcid=0721vQcDls3Ak34dZY1y3h7o&key=77421cf58af4a653e4f55f04cf114492e73a17a2a7d56a0e523c62f16c003b19cdab0cf3a902023d7cbe2af60a58c71d&ascene=0&uin=MjAyNzY1NTU%3D&devicetype=iMac+MacBookPro12%2C1+OSX+OSX+10.11.3+build(15D21)&version=11020201&pass_ticket=ihQKTSTYwhIquv1%2B6HyhJs3I0vZz0qtIoTVci3l%2BikU%3D
