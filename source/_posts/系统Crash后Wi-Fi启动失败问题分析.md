---
title: 系统Crash后Wi-Fi启动失败问题分析
date: 2016-05-20 05:48:38
tags:
- wifi
- android
---
这个问题一开始报过来只有Wi-Fi启动失败  
具体表现为点击开关，一直没反应，重复开启也没用，重启后正常  
之前比较常见的是system image和boot image版本不匹配导致kernel无法加载wifi驱动  
但是从当前问题的log看不是上面的情况，有效log不足以定位具体异常代码，又不知道复现步骤  
无从下手，怎么办？  
先来了解一下Wi-Fi的代码架构

## Wi-Fi Construction
![Wi-Fi架构图](/uploads/wifi_structure.png)  
Wi-Fi的整体通信架构非常简单，在android系统服务中也很典型  
大体就是
1. Appication通过WifiManager和System Server进程中的wifi service通信  
2. wifi service再通过socket和native进程wpa_supplicant通信  
3. 最后由wpa_supplicant向驱动发送命令  

其中第一步IPC不管AsyncChannel还是AIDL都是通过Binder实现的  
我理解各种IPC方式的关系是这样的  
* Binder是以方法调用的形式进行IPC
* AIDL封装Binder省去Binder的固定写法
* Messenger是基于AIDL实现以发消息的形式进行IPC
* AsyncChannel封装了Messenger  

Messenger和AIDL结合Service的使用模板可以参考我的[这篇文章](https://andkid.github.io/2015/10/15/service/)

第二步socket通信的具体代码在wifi.c的wifi_connect_on_socket_path函数  
其创建了两个socket，一个用于向wpa_supplicant发消息，另一个用于监听wpa_supplicant的消息

## Analysis Procedure
### log分析
从log中能看到异常的是这句
`Failed to start supplicant!`  
结合代码
```java
class InitialState extends State {
    @Override
    public void enter() {
        WifiNative.stopHal();
        mWifiNative.unloadDriver();
        ...
    }
    @Override
    public boolean processMessage(Message message) {
        logStateAndMessage(message, getClass().getSimpleName());
        switch (message.what) {
            case CMD_START_SUPPLICANT:
                if (mWifiNative.loadDriver()) {
                    ...
                   /* Stop a running supplicant after a runtime restart
                    * Avoids issues with drivers that do not handle interface down
                    * on a running supplicant properly.
                    */
                    mWifiMonitor.killSupplicant(mP2pSupported);

                    if (WifiNative.startHal() == false) {
                        /* starting HAL is optional */
                        loge(&quot;Failed to start HAL&quot;);
                    }

                    if (mWifiNative.startSupplicant(mP2pSupported)) {
                        setWifiState(WIFI_STATE_ENABLING);
                        if (DBG) log(&quot;Supplicant start successful&quot;);
                        mWifiMonitor.startMonitoring();
                        transitionTo(mSupplicantStartingState);
                    } else {
                        loge(&quot;Failed to start supplicant!&quot;);
                    }
                } else {
                    loge(&quot;Failed to load driver&quot;);
                }
                break;
```
我刚才说的比较常见的驱动加载失败的情况，应该是打出`Failed to load driver`这句log  
而现在可以看出是startSupplicant失败了  
看到这就比较疑惑了，驱动已经加载成功了，而startSupplicant又是启动wifi的必经之路，不会有问题才对  
再结合上下文看到之前有系统异常重启的log，注意，这儿的重启仅仅是zygote进程重新启动不是整个系统reboot  
很开心，丢给导致system crash的模块负责人就完事了^_^

### 场景分析
#### 模拟system crash
很显然，这个问题还没有结束，不能每次system crash就会导致wifi开启不了吧  
从log可以看到system crash前wifi是处于开启状态的  
打开wifi，利用`adb shell stop;adb shell start`来模拟system crash看会不会发生异常  
很遗憾，并没有发生异常，wifi开启非常顺利  
但是，log依然包含`Failed to start supplicant!`  
至少能知道，在system crash后startSupplicant肯定是存在问题的，和复现的场景差别在哪  
差别在scan always的设置，即StaDisabledWithScanState
#### Wi-Fi的StaDisabledWithScanState状态
介绍一下wifi的StaDisabledWithScanState  
该状态下即使Settings里面wifi开关关闭，wpa_supplicant进程已然存在，以便向第三方应用提供扫描结果  
刚才没有复现，是因为虽然一开始失败了，但是由于scan always开启，所以后续还有一次startSupplicant动作并成功了  
关闭该设置开关，Location-&gt;menu-&gt;scan-&gt;Wi-Fi scanning
再次尝试`adb shell stop;adb shell start`  
问题复现了！  
这儿来解释一个问题，为什么打开scan always后续执行一次startSupplicant能成功而出现异常情况多次点击开关都无效  
原因是出现异常情况WifiController的状态机进入了StaEnabledState，即便点击开关也走不到startSupplicant

### 代码分析
回顾刚才WifiStateMachine的代码
```java
/* Stop a running supplicant after a runtime restart
 * Avoids issues with drivers that do not handle interface down
 * on a running supplicant properly.
 */
 mWifiMonitor.killSupplicant(mP2pSupported);

...

 if (mWifiNative.startSupplicant(mP2pSupported)) {
     setWifiState(WIFI_STATE_ENABLING);
     if (DBG) log(&quot;Supplicant start successful&quot;);
     mWifiMonitor.startMonitoring();
     transitionTo(mSupplicantStartingState);
 } else {
     loge(&quot;Failed to start supplicant!&quot;);
 }
```
这儿的逻辑是先`killSupplicant`然后`startSupplicant`
#### killSupplicant
最终走到wifi.c的wifi_stop_supplicant
```java
int wifi_stop_supplicant(int p2p_supported)
{
...
    property_set(&quot;ctl.stop&quot;, supplicant_name); //通过设置ctl.stop的系统属性来启动supplicant
    sched_yield();

    while (count-- &gt; 0) {
        if (property_get(supplicant_prop_name, supp_status, NULL)) { //stop成功后supplicant_prop_name也就是init.svc.p2p_supplicant对应的value会被设为stopped
            if (strcmp(supp_status, &quot;stopped&quot;) == 0) //此处没加块括号
                wifi_stop_fstman(0); //高通工程师添加了该语句却没在上面的if后加块括号
                return 0; //导致即使if判断为false也会return
        }
        usleep(100000);
    }
...
}
```
#### startSupplicant
最终走到wifi.c的wifi_start_supplicant
```java
int wifi_start_supplicant(int p2p_supported)
{
...
    property_set(&quot;ctl.start&quot;, supplicant_name); //通过设置ctl.start的系统属性启动supplicant
    sched_yield(); //这句大致意思是降低本进程执行优先级，为了让上面的流程能先处理，因为现在后续的动作就是等supplicant启动了

    while (count-- &gt; 0) {
        if (pi == NULL) {
            pi = __system_property_find(supplicant_prop_name); //和stop对应，start supplicant成功后会设init.svc.p2p_supplicant为running
        }
        if (pi != NULL) {
            /*
             * property serial updated means that init process is scheduled
             * after we sched_yield, further property status checking is based on this */
            if (__system_property_serial(pi) != serial) { //每当一个system_proper被修改后，其对应的serial也会修改，这儿就是start失败的关键点了，
                                                          //注意，当执行到这儿，上一次的stop动作还没完成！！！
                __system_property_read(pi, NULL, supp_status); //所以，当serial改变实际上是上一次stop的动作触发的，而init.svc.p2p_supplicant是会被设为stopped
                if (strcmp(supp_status, &quot;running&quot;) == 0) {
                    return 0;
                } else if (strcmp(supp_status, &quot;stopped&quot;) == 0) {
                    return -1; //之后就执行到这儿，返回失败！！！
                }
            }
        }
        usleep(100000);
    }
...
```
## 解决方案
给if语句块加上大括号
```java
if (strcmp(supp_status, &quot;stopped&quot;) == 0) {
    wifi_stop_fstman(0);
    return 0;
}
```
