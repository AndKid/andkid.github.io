---
title: Android Dump
date: 2015-10-25 15:21:42
tags:
- android
- dump
- shell
---
**android在shell环境下执行的命令都对应相应的程序实现，下面讲的是如何获取命令对应输出。**

以adb shell dumpsys meminfo 【pid】为例
>注：以下两种方法都需要程序获得权限android.permission.DUMP，而且该 permission在 android.Manifest.permission中说明
Allows an application to retrieve state dump information from system services.
Not for use by third-party applications.

所以，需要将apk push到/system/priv-app/目录下。
***
方法一,忽略该命令方法执行流程，利用
```java
    try {
        Process p = Runtime.getRuntime().exec("dumpsys meminfo " + android.os.Process.myPid());
        BufferedReader br = new BufferedReader(new InputStreamReader(p.getInputStream()));
        String line = null;
        while((line = br.readLine()) != null) {
            System.out.println(line);
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
```
***
方法二，找到源码`frameworks/native/cmds/dumpsys/dumpsys.cpp`
其中代码
```java
sp<IBinder> service = sm->checkService(services[i]);
获取dumpsys命令后对应的注册在ServiceManager中的服务，dumpsys meminfo就是 名字为meminfo的服务，可以用ServiceManager.getService("meminfo")得到。
此处service为Binder的代理对象，也就是说注册在ServiceManager中的服务是个 Binder对象，即 MemBinder，继承了Binder。
int err = service->dump(STDOUT_FILENO, args);
这句就是调用了meminfo对应的MemBinder的Binder本地对象的dump方 法，MemBinder是在 ActivityManagerService中调用
ServiceManager.addService("meminfo", new MemBinder(m))注册;
参数STDOUT_FILENO是标准输出的fd，所以命令会输出到控制台窗口，args就是参 数【pid】等
dump方法中会调用对应pid的ProcessRecord中描述Client应用程序 ApplicationThread的代理对象 thread方法
thread.dumpMemInfo(fd, mi, isCheckinRequest, dumpFullDetails, dumpDalvik, innerArgs);
```
*注：需要调用Client应用程序(相对SystemServer而言)dumpMemInfo方法是因为一些用户空间内存信息Heap Size/Heap Alloc/Heap Free需要在本进程调用相应方法
综上流程，我们只要利用ServceManager.getService("meminfo“)获得MemBinder代理对象再调用dump方法，第一个参数改成自己创建文件对应的FileDescriptor，
第二个参数new String["[pid]"]即可。*

*注：dumpsys meminfo命令需要添加权限，并push到priv-app目录，其他命令依具体情况而言。*
