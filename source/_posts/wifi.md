---
title: Android Wi-Fi Introduction
date: 2015-11-28 12:06:33
tags:
- wifi
- android
---
# WIFI Architecture Overview

>Wi-Fi(Wireless Fidelity)是一个无线网络通信技术的品牌，由Wi-Fi联盟（Wi-Fi Alliance，缩写为WFA）拥有。  
WFA专门负责Wi-Fi认证与商标授权工作。严格地说，Wi-Fi是一个认证的名称，该认证用于测试无线网络设备是否符合IEEE802.11系列协议规范。  
通过该认证的设备将被授予一个名为Wi-Fi CERTIFIED的商标，其目的就是为了保证无线设备之间的互通性。  
不过，随着获得Wi-Fi认证设备的普及，人们也就习以为常地称无线网络为Wi-Fi网络了。

***

## 相关名词解释
* LAN(Local Area Network)：局域网，是路由和主机组成的内部局域网，一般为有线网络
* WAN(Wild Area Network)：广域网，是外部一个更大的局域网
* WLAN(Wireless Local Area Network)：无线局域网，指应用无线通信技术将计算机设备互联起来，构成可以互相通信和实现资源共享的网络体系
* IEEE802.11协议规范：由IEEE协会针对无线局域网通信在网络通信协议层中的MAC(Media Access Control)和PHY(Physical Layer)两层制定的规范
* AP（Access Point）：无线访问接入点，将各个无线网络客户端连接到一起并接入以太网
* SSID：标识一个无线网络
* Band：频率范围，一般ap支持5g或2.4g两个频率范围的无线信号
* Channel：信道，对频段的进一步划分
* Channel Width：信道宽度，即信道带宽，影响吞吐量
* RTS/CTS：通信握手协议，一旦待传送数据大于设置的上限字节数就启动该协议，以解决“隐藏终端”问题
* WPA/EAP：加密认证协议
* RSSI(Receive Signal Strength Indication)：接收信号强度指示，反映了无线网络质量的好坏


***

## 代码结构
### 应用层
/packages/apps/Settings/wifi
* WifiSettings.java  
    用户对wifi功能的直接控制，触发相关流程开启，包括wifi启用/关闭，连接/断开，扫描等；  
    通过注册receiver接收底层上传的wifi状态变化消息，并在UI上体现出来。

* WifiEnable.java  
    通过控制switch bar，调用WifiManager对应接口发送命令给底层。

***

### Framework层
/frameworks/opt/net/wifi/service  
/frameworks/base/wifi/java/android/net/wifi
* WifiManager.java  
    Wifi Client端提供了管理wifi连接的主要接口，通过`Context.getSystemService(Context.WIFI_SERVICE)`获得实例。  
    用于管理已配置的网络列表，获取当前处于连接状态的网络信息。

* WifiServiceImpl.java  
    实现了WifiManager对wifi远程操作的请求。Framework层的核心控制类。

* WifiStateMachine.java  
    跟踪wifi连接状态，初始化各wifi状态并处理相应的事件。  
    wifi支持三种模式：Client，SoftAp和p2p，WifiStateMachine处理Client和SoftAp的动作，WifiP2pService处理p2p的动作。

* WifiMonitor.java  
    监听wpa_supplicant服务端的事件并传递给相应StateMachine处理。

* WifiNative.java  
    调用本地方法启用/终止wpa_supplicant守护进程并向其发送请求。  
    WifiMonitor监听wpa_supplicant事件的线程调用了WifiNative的`waitForEvent()`方法。

* com_android_server_wifi_WifiNative.cpp  
    通过JNI调用本地函数，实现了对wpa_supplicant硬件抽象层函数调用的封装。

***

### HAL层
/hardware/libhardware_legacy/wifi/
* wifi.c  
    wifi硬件抽象层部分，与wpa_supplicant守护进程通信，实现了驱动加载、wpa_supplicant启动、命令发送、消息监听等功能。  
    通过insmod加载驱动，通过android属性系统的`property_set("ctl.start", supplicant_name)`启动wpa_supplicant，  
    通过包含wpa_ctrl.h头文件函数实现wpa_ctrl_request发送命令等功能与wpa_supplicant通信。

***

### wpa_supplicant
/external/wpa_supplicant_8/
* wpa_ctrl.c
    提供给外部程序与wpa_supplicant进程通信的接口，由socket实现跨进程通信。
    主要方法有wpa_ctrl_open()打开连接，wpa_ctrl_request()发送命令，wpa_ctrl_recv()接收消息。  
