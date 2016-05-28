---
title: Android Wi-Fi Framework
date: 2016-01-02 18:09:58
tags:
- wifi
- android
---
# Wi-Fi Framework

WifiService是Wi-Fi功能的主入口，下面分析initialize，enable和connect流程。

## Initialize
在SystemServer中调用mSystemServiceManager.startService(WIFI_SERVICE_CLASS)创建WifiService并将其添加到Service列表  
具体实现类WifiServiceImpl的构造方法
```java
public WifiServiceImpl(Context context) {
    mContext = context;
    mInterfaceName =  SystemProperties.get("wifi.interface", "wlan0");
    mTrafficPoller = new WifiTrafficPoller(mContext, mInterfaceName);
    mWifiStateMachine = new WifiStateMachine(mContext, mInterfaceName, mTrafficPoller);//创建WifiStateMachine
    mWifiStateMachine.enableRssiPolling(true);//RSSI为信号强度指示
    mBatteryStats = BatteryStatsService.getService();
    mAppOps = (AppOpsManager)context.getSystemService(Context.APP_OPS_SERVICE);
    mNotificationController = new WifiNotificationController(mContext, mWifiStateMachine);
    mSettingsStore = new WifiSettingsStore(mContext);
    HandlerThread wifiThread = new HandlerThread("WifiService");//创建HandlerThread用于接收并处理消息
    wifiThread.start();
    mClientHandler = new ClientHandler(wifiThread.getLooper());//mClientHandler用于接收WifiManager传过来的消息
    mWifiStateMachineHandler = new WifiStateMachineHandler(wifiThread.getLooper());//mWifiStateMachineHandler接收WifiStateMachine传过来的消息
    mWifiController = new WifiController(mContext, this, wifiThread.getLooper());
    mBatchedScanSupported = mContext.getResources().getBoolean(
            R.bool.config_wifi_batched_scan_supported);
}
```
WifiStateMachine在构造时  
第一阶段创建WifiNative，WifiMonitor，SupplicantStateTracker等  
```java
public WifiStateMachine(Context context, String wlanInterface,
        WifiTrafficPoller trafficPoller){
    super("WifiStateMachine");
    mContext = context;
    mInterfaceName = wlanInterface;
    mNetworkInfo = new NetworkInfo(ConnectivityManager.TYPE_WIFI, 0, NETWORKTYPE, "");//代表一个网络设备的状态信息
    mBatteryStats = IBatteryStats.Stub.asInterface(ServiceManager.getService(
            BatteryStats.SERVICE_NAME));

    IBinder b = ServiceManager.getService(Context.NETWORKMANAGEMENT_SERVICE);
    mNwService = INetworkManagementService.Stub.asInterface(b);//通过“netd”socket和netd守护进程通信

    mP2pSupported = mContext.getPackageManager().hasSystemFeature(
            PackageManager.FEATURE_WIFI_DIRECT);

    mWifiNative = new WifiNative(mInterfaceName);//和wpa_supplicant通信
    mWifiConfigStore = new WifiConfigStore(context, mWifiNative);//对应/data/misc/wifi/ipconfig.txt配置文件，Settngs中无线网络高级选项的设置
    mWifiAutoJoinController = new WifiAutoJoinController(context, this,
            mWifiConfigStore, mWifiConnectionStatistics, mWifiNative);
    mWifiMonitor = new WifiMonitor(this, mWifiNative);//内部创建线程并借助WifiNative取接收并处理来自WPAS的信息
    mWifiInfo = new WifiInfo();
    mSupplicantStateTracker = new SupplicantStateTracker(context, this, mWifiConfigStore,
            getHandler());//跟踪WPAS状态信息，也是StateMachine
    mLinkProperties = new LinkProperties();//描述网络链接属性，IP地址、DNS地址和路由设置等

    IBinder s1 = ServiceManager.getService(Context.WIFI_P2P_SERVICE);
    mWifiP2pServiceImpl = (WifiP2pServiceImpl)IWifiP2pManager.Stub.asInterface(s1);

    mNetworkInfo.setIsAvailable(false);
    mLastBssid = null;
    mLastNetworkId = WifiConfiguration.INVALID_NETWORK_ID;
    mLastSignalLevel = -1;

    mNetlinkTracker = new NetlinkTracker(mInterfaceName, new NetlinkTracker.Callback() {
        public void update(LinkProperties lp) {
            sendMessage(CMD_UPDATE_LINKPROPERTIES, lp);
        }
    });
    try {
        mNwService.registerObserver(mNetlinkTracker);
    } catch (RemoteException e) {
        loge("Couldn't register netlink tracker: " + e.toString());
    }

    mAlarmManager = (AlarmManager)mContext.getSystemService(Context.ALARM_SERVICE);
    mScanIntent = getPrivateBroadcast(ACTION_START_SCAN, SCAN_REQUEST);
    mBatchedScanIntervalIntent = getPrivateBroadcast(ACTION_REFRESH_BATCHED_SCAN, 0);

    // Make sure the interval is not configured less than 10 seconds
    int period = mContext.getResources().getInteger(
            R.integer.config_wifi_framework_scan_interval);
    if (period < frameworkMinScanIntervalSaneValue) {
        period = frameworkMinScanIntervalSaneValue;
    }
    mDefaultFrameworkScanIntervalMs = period;
    Settings.Global.putInt(mContext.getContentResolver(),
        Settings.Global.WIFI_SCAN_INTERVAL_WHEN_P2P_CONNECTED_MS,
        mDefaultDisconnectedScanIntervelWhenP2pConnected);
    mDriverStopDelayMs = mContext.getResources().getInteger(
            R.integer.config_wifi_driver_stop_delay);

    mBackgroundScanSupported = mContext.getResources().getBoolean(
            R.bool.config_wifi_background_scan_support);

    mPrimaryDeviceType = mContext.getResources().getString(
            R.string.config_wifi_p2p_device_type);

    //WIFI_SUSPEND_OPTIMIZATIONS_ENABLED用于控制手机睡眠期间是否保持Wi-Fi开启
    mUserWantsSuspendOpt.set(Settings.Global.getInt(mContext.getContentResolver(),
                Settings.Global.WIFI_SUSPEND_OPTIMIZATIONS_ENABLED, 1) == 1);

    mNetworkCapabilitiesFilter.addTransportType(NetworkCapabilities.TRANSPORT_WIFI);
    mNetworkCapabilitiesFilter.addCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET);
    mNetworkCapabilitiesFilter.addCapability(NetworkCapabilities.NET_CAPABILITY_NOT_RESTRICTED);
    mNetworkCapabilitiesFilter.setLinkUpstreamBandwidthKbps(1024 * 1024);
    mNetworkCapabilitiesFilter.setLinkDownstreamBandwidthKbps(1024 * 1024);
    // TODO - needs to be a bit more dynamic
    mNetworkCapabilities = new NetworkCapabilities(mNetworkCapabilitiesFilter);

    ......//处理扫描、屏幕亮灭等广播事件

}
```
重点介绍WifiNative、WifiMonitor、SupplicantStateTracker  
##### WifiNative
对应JNI文件是com_android_server_wifi_WifiNative.cpp  
主要讲两个方法  
第一个startSupplicant方法通过JNI调用到hardware/libhardware_legacy/wifi/wifi.c
```c
int wifi_start_supplicant(int p2p_supported)
{
    char supp_status[PROPERTY_VALUE_MAX] = {'\0'};
    int count = 200; /* wait at most 20 seconds for completion */
#ifdef HAVE_LIBC_SYSTEM_PROPERTIES
    const prop_info *pi;
    unsigned serial = 0, i;
#endif

    if (p2p_supported) {
        strcpy(supplicant_name, P2P_SUPPLICANT_NAME);
        strcpy(supplicant_prop_name, P2P_PROP_NAME);

        /* Ensure p2p config file is created */
        if (ensure_config_file_exists(P2P_CONFIG_FILE) < 0) {
            ALOGE("Failed to create a p2p config file");
            return -1;
        }

    } else {
        strcpy(supplicant_name, SUPPLICANT_NAME);
        strcpy(supplicant_prop_name, SUPP_PROP_NAME);
    }

    /* Check whether already running */
    if (property_get(supplicant_prop_name, supp_status, NULL)
            && strcmp(supp_status, "running") == 0) {
        return 0;
    }

    /* Before starting the daemon, make sure its config file exists */
    if (ensure_config_file_exists(SUPP_CONFIG_FILE) < 0) {
        ALOGE("Wi-Fi will not be enabled");
        return -1;
    }

    if (ensure_entropy_file_exists() < 0) {
        ALOGE("Wi-Fi entropy file was not created");
    }

    /* Clear out any stale socket files that might be left over. */
    wpa_ctrl_cleanup();

    /* Reset sockets used for exiting from hung state */
    exit_sockets[0] = exit_sockets[1] = -1;

#ifdef HAVE_LIBC_SYSTEM_PROPERTIES
    /*
     * Get a reference to the status property, so we can distinguish
     * the case where it goes stopped => running => stopped (i.e.,
     * it start up, but fails right away) from the case in which
     * it starts in the stopped state and never manages to start
     * running at all.
     */
    pi = __system_property_find(supplicant_prop_name);
    if (pi != NULL) {
        serial = __system_property_serial(pi);
    }
#endif
    property_get("wifi.interface", primary_iface, WIFI_TEST_INTERFACE);

    property_set("ctl.start", supplicant_name);
    sched_yield();

    while (count-- > 0) {
#ifdef HAVE_LIBC_SYSTEM_PROPERTIES
        if (pi == NULL) {
            pi = __system_property_find(supplicant_prop_name);
        }
        if (pi != NULL) {
            /*
             * property serial updated means that init process is scheduled
             * after we sched_yield, further property status checking is based on this */
            if (__system_property_serial(pi) != serial) {
                __system_property_read(pi, NULL, supp_status);
                if (strcmp(supp_status, "running") == 0) {
                    return 0;
                } else if (strcmp(supp_status, "stopped") == 0) {
                    return -1;
                }
            }
        }
#else
        if (property_get(supplicant_prop_name, supp_status, NULL)) {
            if (strcmp(supp_status, "running") == 0)
                return 0;
        }
#endif
        usleep(100000);
    }
    return -1;
}
```
最重要的是’property_set("ctl.start", supplicant_name)‘来启动位于/system/bin/wpa_supplicant的可执行程序

另一个connectToSupplicant方法在Wifi.c中实现
```c
int wifi_connect_on_socket_path(const char *path)
{
    char supp_status[PROPERTY_VALUE_MAX] = {'\0'};

    /* Make sure supplicant is running */
    if (!property_get(supplicant_prop_name, supp_status, NULL)
            || strcmp(supp_status, "running") != 0) {
        ALOGE("Supplicant not running, cannot connect");
        return -1;
    }

    ctrl_conn = wpa_ctrl_open(path);//用于向WPAS发送命令
    if (ctrl_conn == NULL) {
        ALOGE("Unable to open connection to supplicant on \"%s\": %s",
             path, strerror(errno));
        return -1;
    }
    monitor_conn = wpa_ctrl_open(path);//用于从WPAS接收命令
    if (monitor_conn == NULL) {
        wpa_ctrl_close(ctrl_conn);
        ctrl_conn = NULL;
        return -1;
    }
    if (wpa_ctrl_attach(monitor_conn) != 0) {
        wpa_ctrl_close(monitor_conn);
        wpa_ctrl_close(ctrl_conn);
        ctrl_conn = monitor_conn = NULL;
        return -1;
    }

    if (socketpair(AF_UNIX, SOCK_STREAM, 0, exit_sockets) == -1) {
        wpa_ctrl_close(monitor_conn);
        wpa_ctrl_close(ctrl_conn);
        ctrl_conn = monitor_conn = NULL;
        return -1;
    }

    return 0;
}

/* Establishes the control and monitor socket connections on the interface */
int wifi_connect_to_supplicant()
{
    static char path[PATH_MAX];

    if (access(IFACE_DIR, F_OK) == 0) {
        snprintf(path, sizeof(path), "%s/%s", IFACE_DIR, primary_iface);
    } else {
        snprintf(path, sizeof(path), "@android:wpa_%s", primary_iface);//path为@android:wpa_wlan0
    }
    return wifi_connect_on_socket_path(path);
}
```
wifi.c中wifi_send_command会用ctrl_conn中的wpa_ctrl对象想WPAS发送命令并接收回复  
wifi_ctrl_recv函数将使用monitor_conn中的wpa_ctrl对象接收来自WPAS的消息  

##### WifiMonitor
WifiMonitor最重要的是其内部的WifiMonitor线程，专门用于接收来自WPAS的消息  
```java
private static class MonitorThread extends Thread {
    private final WifiNative mWifiNative;
    private final WifiMonitorSingleton mWifiMonitorSingleton;

    public MonitorThread(WifiNative wifiNative, WifiMonitorSingleton wifiMonitorSingleton) {
        super("WifiMonitor");
        mWifiNative = wifiNative;
        mWifiMonitorSingleton = wifiMonitorSingleton;
    }

    public void run() {
        Log.d(TAG, "MonitorThread start with mConnected=" + mWifiMonitorSingleton.mConnected);

        //noinspection InfiniteLoopStatement
        for (;;) {
            if (!mWifiMonitorSingleton.mConnected) {
                Log.d(TAG, "MonitorThread exit because mConnected is false");
                break;
            }

            String eventStr = mWifiNative.waitForEvent();//waitForEvent内部会调用wifi.c中的wifi_wait_on_socket函数

            // Skip logging the common but mostly uninteresting scan-results event
            if (DBG && eventStr.indexOf(SCAN_RESULTS_STR) == -1) {
                Log.d(TAG, "Event [" + eventStr + "]");
            }

            if (mWifiMonitorSingleton.dispatchEvent(eventStr)) {
                Log.d(TAG, "Disconnecting from the supplicant, no more events");
                break;
            }
        }
    }
}
```
```c
int wifi_wait_on_socket(char *buf, size_t buflen)
{
    size_t nread = buflen - 1;
    int result;
    char *match, *match2;

    if (monitor_conn == NULL) {
        return snprintf(buf, buflen, "IFNAME=%s %s - connection closed",
                        primary_iface, WPA_EVENT_TERMINATING);
    }

    result = wifi_ctrl_recv(buf, &nread);//监听monitor_conn中的socket

    /* Terminate reception on exit socket */
    if (result == -2) {
        return snprintf(buf, buflen, "IFNAME=%s %s - connection closed",
                        primary_iface, WPA_EVENT_TERMINATING);
    }

    if (result < 0) {
        ALOGD("wifi_ctrl_recv failed: %s\n", strerror(errno));
        return snprintf(buf, buflen, "IFNAME=%s %s - recv error",
                        primary_iface, WPA_EVENT_TERMINATING);
    }
    buf[nread] = '\0';
    /* Check for EOF on the socket */
    if (result == 0 && nread == 0) {
        /* Fabricate an event to pass up */
        ALOGD("Received EOF on supplicant socket\n");
        return snprintf(buf, buflen, "IFNAME=%s %s - signal 0 received",
                        primary_iface, WPA_EVENT_TERMINATING);
    }
    /*
     * Events strings are in the format
     *
     *     IFNAME=iface <N>CTRL-EVENT-XXX
     *        or
     *     <N>CTRL-EVENT-XXX
     *
     * where N is the message level in numerical form (0=VERBOSE, 1=DEBUG,
     * etc.) and XXX is the event name. The level information is not useful
     * to us, so strip it off.
     */

    if (strncmp(buf, IFNAME, IFNAMELEN) == 0) {
        match = strchr(buf, ' ');
        if (match != NULL) {
            if (match[1] == '<') {
                match2 = strchr(match + 2, '>');
                if (match2 != NULL) {
                    nread -= (match2 - match);
                    memmove(match + 1, match2 + 1, nread - (match - buf) + 1);
                }
            }
        } else {
            return snprintf(buf, buflen, "%s", WPA_EVENT_IGNORE);
        }
    } else if (buf[0] == '<') {
        match = strchr(buf, '>');
        if (match != NULL) {
            nread -= (match + 1 - buf);
            memmove(buf, match + 1, nread + 1);
            ALOGV("supplicant generated event without interface - %s\n", buf);
        }
    } else {
        /* let the event go as is! */
        ALOGW("supplicant generated event without interface and without message level - %s\n", buf);
    }

    return nread;
}
```
handleEvent方法
```java
/**
 * Handle all supplicant events except STATE-CHANGE
 * @param event the event type
 * @param remainder the rest of the string following the
 * event name and &quot;&#8195;&#8212;&#8195;&quot;
 */
void handleEvent(int event, String remainder) {
    if (DBG) {
        logDbg("handleEvent " + Integer.toString(event) + "  " + remainder);
    }
    switch (event) {
        case DISCONNECTED:
            handleNetworkStateChange(NetworkInfo.DetailedState.DISCONNECTED, remainder);
            break;

        case CONNECTED:
            handleNetworkStateChange(NetworkInfo.DetailedState.CONNECTED, remainder);
            break;

        case SCAN_RESULTS:
            mStateMachine.sendMessage(SCAN_RESULTS_EVENT);
            break;

        case UNKNOWN:
            if (DBG) {
                logDbg("handleEvent unknown: " + Integer.toString(event) + "  " + remainder);
            }
            break;
        default:
            break;
    }
}
```

##### SupplicantStateTracker
WPAS的状态机
```java
public SupplicantStateTracker(Context c, WifiStateMachine wsm, WifiConfigStore wcs, Handler t) {
    super(TAG, t.getLooper());

    mContext = c;
    mWifiStateMachine = wsm;
    mWifiConfigStore = wcs;
    mBatteryStats = (IBatteryStats)ServiceManager.getService(BatteryStats.SERVICE_NAME);
    addState(mDefaultState);
        addState(mUninitializedState, mDefaultState);
        addState(mInactiveState, mDefaultState);
        addState(mDisconnectState, mDefaultState);
        addState(mScanState, mDefaultState);
        addState(mHandshakeState, mDefaultState);
        addState(mCompletedState, mDefaultState);
        addState(mDormantState, mDefaultState);

    setInitialState(mUninitializedState);
    setLogRecSize(50);
    setLogOnlyTransitions(true);
    //start the state machine
    start();
}
```

第二阶段添加状态并设置初始化状态
```java
addState(mDefaultState);
  addState(mInitialState, mDefaultState);
    addState(mSupplicantStartingState, mDefaultState);
    addState(mSupplicantStartedState, mDefaultState);
        addState(mDriverStartingState, mSupplicantStartedState);
        addState(mDriverStartedState, mSupplicantStartedState);
            addState(mScanModeState, mDriverStartedState);
            addState(mConnectModeState, mDriverStartedState);
                addState(mL2ConnectedState, mConnectModeState);
                    addState(mObtainingIpState, mL2ConnectedState);
                    addState(mVerifyingLinkState, mL2ConnectedState);
                    addState(mConnectedState, mL2ConnectedState);
                    addState(mRoamingState, mL2ConnectedState);
                addState(mDisconnectingState, mConnectModeState);
                addState(mDisconnectedState, mConnectModeState);
                addState(mWpsRunningState, mConnectModeState);
        addState(mWaitForP2pDisableState, mSupplicantStartedState);
        addState(mDriverStoppingState, mSupplicantStartedState);
        addState(mDriverStoppedState, mSupplicantStartedState);
    addState(mSupplicantStoppingState, mDefaultState);
    addState(mSoftApStartingState, mDefaultState);
    addState(mSoftApStartedState, mDefaultState);
        addState(mTetheringState, mSoftApStartedState);
        addState(mTetheredState, mSoftApStartedState);
        addState(mUntetheringState, mSoftApStartedState);
setInitialState(mInitialState);
```
WifiStateMacine通过mReplyChannel和WifiManager通信

## Enable&Join
点击Wi-Fi开关后会调用WifiEnable的onSwitchChanged
```java
@Override
public void onSwitchChanged(Switch switchView, boolean isChecked) {
    //Do nothing if called as a result of a state machine event
    if (mStateMachineEvent) {
        return;
    }
    // Show toast message if Wi-Fi is not allowed in airplane mode
    if (isChecked && !WirelessSettings.isRadioAllowed(mContext, Settings.Global.RADIO_WIFI)) {
        Toast.makeText(mContext, R.string.wifi_in_airplane_mode, Toast.LENGTH_SHORT).show();
        // Reset switch to off. No infinite check/listenenr loop.
        mSwitchBar.setChecked(false);
        return;
    }

    // Disable tethering if enabling Wifi
    int wifiApState = mWifiManager.getWifiApState();
    if (isChecked && ((wifiApState == WifiManager.WIFI_AP_STATE_ENABLING) ||
            (wifiApState == WifiManager.WIFI_AP_STATE_ENABLED))) {
        mWifiManager.setWifiApEnabled(null, false);
    }

    if (!mWifiManager.setWifiEnabled(isChecked)) {
        // Error
        mSwitchBar.setEnabled(true);
        Toast.makeText(mContext, R.string.wifi_error, Toast.LENGTH_SHORT).show();
    }
}
```
WifiManager的setWifiEnabled函数将触发WifiService展开一系列的动作，在此期间，WifiService会发送广播  
WifiSettings会接收这些广播并更新UI
```java
public WifiSettings() {
    super(DISALLOW_CONFIG_WIFI);
    mFilter = new IntentFilter();
    mFilter.addAction(WifiManager.WIFI_STATE_CHANGED_ACTION);
    mFilter.addAction(WifiManager.SCAN_RESULTS_AVAILABLE_ACTION);
    mFilter.addAction(WifiManager.NETWORK_IDS_CHANGED_ACTION);
    mFilter.addAction(WifiManager.SUPPLICANT_STATE_CHANGED_ACTION);
    mFilter.addAction(WifiManager.CONFIGURED_NETWORKS_CHANGED_ACTION);
    mFilter.addAction(WifiManager.LINK_CONFIGURATION_CHANGED_ACTION);
    mFilter.addAction(WifiManager.NETWORK_STATE_CHANGED_ACTION);
    mFilter.addAction(WifiManager.RSSI_CHANGED_ACTION);

    mReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            handleEvent(intent);
        }
    };

    mScanner = new Scanner(this);
}
```
WifiSettings在接收到WifiManager.WIFI_STATE_CHANGED_ACTION后会调用WifiManager的startScan  
之后会接收到WifiManager.SCAN_RESULTS_AVAILABLE_ACTION，会调用updateAccessPoint  
当WifiSettings界面显示出周围无线网络后，点击一个AP加入会调用onPreferenceTreeClick方法弹出对话框  
输入密码后点击连接，会调用WifiManager的connect方法，此时WifiSettings工作告一段落，后续工作就是等待并处理广播事件  
如果一切顺利，它将接收一个NETWORK_STATE_CHANGED_ACTION广播事件以告知连接成功  
此间涉及到的setWifiEnabled/startScan/connect会利用WifiNative与WPAS通信，WifiMonitor接收到消息传递给WifiStateMacine切换状态
