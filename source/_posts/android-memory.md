---
title: Android Memory
date: 2015-11-01 10:28:53
tags:
- android
- memory
---
### java
* Debug.getHeapSize()			获取当前进程占用Native堆内存的大小，对应命令adb shell dumpsys info [pid]中Native Heap->Heap Size  
* Debug.getNativeHeapFreeSize()		获取当前进程可用Native堆内存的大小，对应命令adb shell dumpsys info [pid]中Native Heap->Heap Free  
* Debug.getNativeHeapAllocatedSize()	获取当前进程已分配Native堆内存的大小，对应命令adb shell dumpsys info [pid]中Native Heap->Heap Alloc，等于上两项相减  
* Runtime: totalMemory()			获取当前进程占用Dalvik堆内存的大小，对应命令adb shell dumpsys info [pid]中Dalvik Heap->Heap Size  
* Runtime: freeMemory()			获取当前进程可用Dalvik堆内存的大小，对应命令adb shell dumpsys info [pid]中Dalvik Heap->Heap Free  
* ActivityManager: getMemoryClass()	获取每个进程（不是应用）在Dalvik+Native内存中最多分配的大小，超出抛OOM exception，其实在合适的情况添加属性android:process=":remote"可让组件跑在另一个进程（加了‘：’为私有进程，其他应用不能把组件加进来），也就可以申请更多的内存  
* Runtime: maxMemory()			获取每个进程（不是应用）在Dalvik内存中最多分配的大小（试了多台手机和上面的值是一样的具体限制Dalvik还是both不确定）  
* ActivityManager: MemoryInfo[] getProcessMemoryInfo(int[] pids)根据进程获得Debug.MemoryInfo，其中有dalvikPrivateDirty等变量  

### shelll
adb shell procrank
* VSS - Virtual Set Size 虚拟耗用内存（包含共享库占用的内存）
* RSS - Resident Set Size 实际使用物理内存（包含共享库占用的内存）
* PSS - Proportional Set Size 实际使用的物理内存（比例分配共享库占用的内存）
* USS - Unique Set Size 进程独自占用的物理内存（不包含共享库占用的内存）
VSS和RSS没什么用，PSS把内存共享部分分摊给各进程，PSS和USS应该分别对应adb shell dumpsys meminfo [pid]中的Pss Total和Private Dirty但是两个得到内存信息机制不同，数据有偏差
***
对于计算一个进程内存比较有用的：
* 虚拟内存
Debug.getNativeHeapAllocatedSize()
Runtime: totalMemory()-Runtime: freeMemory()
* 物理内存
Debug.MemoryInfo中变量dalvikPrivateDirty、nativePrivateDirty、otherPrivateDirty
