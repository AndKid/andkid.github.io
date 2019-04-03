---
title: 安卓性能优化套路
date: 2018-05-09 20:08:54
tags:
- performance
- android
---
网上研究性能问题的文章很多，有介绍原理的，有介绍分析工具的，有讲具体问题点的，还有一堆山寨的  
本文是在吸纳了所有的精髓后，总结的套路，旨在真正定位并解决性能问题  

## 发现问题
性能问题说白了，就是程序卡顿从而影响用户体验，而卡顿的直接原因就是掉帧  
在安卓上以FPS等于60（显示一帧的时间间隔16.7ms）为标准，低于这个标准即发生了掉帧  
可以从以下方式判断掉帧的严重程度，并以此来验证优化后的效果  

1. 从使用者角度明显感受到卡顿
2. 6.0以上的系统，通过开发者模式中“GPU呈现模式分析”的条形图来看，有大量帧时间超过绿线代表的16.7ms
![](/uploads/profile_gpu_rendering_graph.png)
3. 6.0以上的系统，通过开发者模式中“GPU呈现模式分析”的gfxinfo选项，  
通过命令`adb shell "dumpsys gfxinfo com.fanli.android.apps reset"`来获取两次命令之间的所有帧状态，  
显示结果中的“Total frames rendered”和“Janky frames”可以看出掉帧率  

## 16.7ms内需要执行的任务
显示一帧图像到屏幕只有短短的16.7ms，期间CPU和GPU需要做很多事情，简单了解整个流程有助于针对性地找出影响各个阶段的耗时因素

* CPU做业务处理/响应用户输入
	+ 针对UI线程
* measure/layout
	+ 在view创建或被listview/recycleview复用的时候会调用，滚动过程中不会调用
* draw(create display list)
	+ display list会被系统缓存起来，某些情况的transform/scroll/animation不会重建display list，即不会调draw，调invalidate会重新创建
* upload image
	+ 将image从CPU拷贝到GPU
* issue commands(将命令发送给GPU)
	+ 在将绘制命令发送给GPU前，对命令做一些变换和裁剪的修改
* GPU做栅格化、复制图像缓冲等操作

## 各阶段耗时因素
每个阶段都有可能存在耗时因素，先简单列一下，下面会具体介绍典型因素及如何定位解决

* 主线程处理耗时业务
* measure/layout，view被频繁创建移除，过多调用requestLayout，view层级太深（单次执行耗时+double taxation）
* draw，在onDraw执行耗时操作，创建小对象
* upload image，image太大超出必要尺寸
* issue commands，在draw的时候，小命令太多
* GPU，overdraw现象

## 耗时因素介绍

### Double Taxation
某些容器需要传递两次mearsure/layout才能最终确定布局，比如RelativeLayout，加了layout_weight的LinearLayout  
这是系统默认行为，我们无法避免，只能尽量减少以下情况使用double taxation的容器  
1. root view
2. 当前view下面有较深的层级
3. listview/recyclerview的item

### Overdraw
overdraw指在绘制同一帧时，一个屏幕像素点被重复绘制多次，这样会浪费GPU性能  
可通过减少层级，或去掉不必要的背景色来缓解overdraw  
比如，嵌套的LinearLayout可用RelativeLayout来代替

> 看似double taxation和overdraw问题的处理方式上有些矛盾的地方，double taxation鼓励不用RelativeLayout，overdraw又鼓励用RelativeLayout...  
这两者间有个权衡，如果是double taxation中提到的三种情况就不能用RelativeLayout，因为此时double taxation引起的问题更严重  
如果是个层次不深且不会被频繁创建复用的view，尽量用RelativeLayout来减少层级  

### Upload Image
如果当前页面图片太多，从CPU往GPU同步image内存时会较耗时  
解决方法是，image大小尽量适配view的宽高，图片库在decode线程在decode完成后调用prepareToDraw，提前同步image到GPU

## 定位问题点
![](/uploads/profile_gpu_rendering_graph_legend.png)

![](/uploads/screen_capture_render_indicator_brands.png)

从图可以大致看出来在复用item的时候掉帧比较严重，掉了3帧以上  
根据“GPU呈现模式分析”的条形图色段能大致看出各阶段的耗时程度，下面是具体耗时因素分析  
1. Overdraw  
打开设置中“调试GPU过度绘制”选项，如果红彤彤一大片，说明overdraw现象严重  
	* 真彩色： 没有过度绘制
	* 蓝色： 过度绘制 1 次
	* 绿色： 过度绘制 2 次
	* 粉色： 过度绘制 3 次
	* 红色： 过度绘制 4 次或更多
	![](/uploads/screen_capture_overdraw_brands.png)
2. Measure/Layout  
	* 通过Android Studio内置Layout Inspector查看布局结构，看看有没有可优化的层级嵌套（通过<merge>标签），以及是否存在严重double taxation的情况
	![](/uploads/screen_capture_layout_inspector_brands.png)
	* 通过Hierarchy Viewer查看单个view节点measure/layout的耗时
3. CPU/Memory  
	* 通过Android Studio内置Android Profiler进一步分析耗时方法以及查看是否存在内存抖动

## 参考链接
https://developer.android.com/training/improving-layouts/optimizing-layout.html  
https://developer.android.com/topic/performance/rendering/optimizing-view-hierarchies.html  
https://developer.android.com/topic/performance/rendering/profile-gpu.html  
https://developer.android.com/topic/performance/rendering/overdraw.html  
https://developer.android.com/studio/profile/hierarchy-viewer.html  
https://developer.android.com/studio/profile/inspect-gpu-rendering.html  
https://developer.android.com/studio/profile/cpu-profiler.html  
https://medium.com/google-developers/draw-what-you-see-and-clip-the-e11-out-of-the-rest-6df58c47873e  
https://medium.com/google-developers/simplify-complex-view-hierarchies-5d358618b06f  