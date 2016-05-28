---
title: Android Wi-Fi Supplicant
date: 2016-01-15 15:22:45
tags:
- wifi
- android
---
# Wi-Fi WPA_Supplicant

    service wpa_supplicant /system/bin/wpa_supplicant \
        -iwlan0 -Dnl80211 -c/data/misc/wifi/wpa_supplicant.conf \
        -I/system/etc/wifi/wpa_supplicant_overlay.conf \
        -O/data/misc/wifi/sockets -dd \
        -e/data/misc/wifi/entropy.bin -g@android:wpa_wlan0
        #   we will start as root and wpa_supplicant will switch to user wifi
        #   after setting up the capabilities required for WEXT
        #   user wifi
        #   group wifi inet keystore
        class main
        socket wpa_wlan0 dgram 660 wifi wifi
        disabled
        oneshot

        options:
          -b = optional bridge interface name
          -B = run daemon in the background
          -c = Configuration file
          -C = ctrl_interface parameter (only used if -c is not)
          -i = interface name
          -I = additional configuration file
          -d = increase debugging verbosity (-dd even more)
          -D = driver name (can be multiple drivers: nl80211,wext)
          -e = entropy file
          -g = global ctrl_interface
          -G = global ctrl_interface group
          -K = include keys (passwords, etc.) in debug output
          -t = include timestamp in debug messages
          -h = show this help text
          -L = show license (BSD)
          -o = override driver parameter for new interfaces
          -O = override ctrl_interface parameter for new interfaces
          -p = driver parameters
          -P = PID file
          -q = decrease debugging verbosity (-qq even less)
          -v = show version
          -W = wait for a control interface monitor before starting
          -m = Configuration file for the P2P Device interface
          -N = start describing new interface
        example:
          wpa_supplicant -Dnl80211 -iwlan0 -c/data/misc/wifi/wpa_supplicant.conf


所有来自客户端的命令都由wpa_supplicant_ctrl_iface_receive函数处理

insmod /system/lib/modules/wlan.ko
wpa_supplicant -Dnl80211 -iwlan0 -c/data/misc/wifi/wpa_supplicant.conf -I/system/etc/wifi/wpa_supplicant_overlay.conf -O/data/misc/wifi/sockets -dd -e/data/misc/wifi/entropy.bin
与wpa_supplicant通信的socket，在/data/misc/wifi/wpa_supplicant.conf配置文件的ctrl_interface指定（/data/misc/wifi/sockets）
wpa_cli -p/data/misc/wifi/sockets -i wlan0
driver SETBAND 0//不生效

wpa_driver_nl80211_get_scan_results

src/drivers/driver.h    struct wpa_driver_scan_params

// call wifi native to start the scan
        if (startScanNative(type, freqs)) {//传递了freqs频段
            ...
        }

wpa_s->manual_scan_freqs//指定手动设置的频段

//触发驱动扫描
```
wpa_supplicant_trigger_scan
radio_add_work
```

```
WCNSS_qcom_cfg.ini
#Preferred band (both or 2.4 only or 5 only)
BandCapability=0
```

WifiManager提供了扫描的参数的自定义接口，比如在ScanSettings添加一个freqMHz为5500的WifiChannel来过滤扫描结果

```java
WifiManager.setFrequencyBand
/**
     * Set the operational frequency band.
     * @param band  One of
     *     {@link #WIFI_FREQUENCY_BAND_AUTO},0
     *     {@link #WIFI_FREQUENCY_BAND_5GHZ},1
     *     {@link #WIFI_FREQUENCY_BAND_2GHZ},2
     * @param persist {@code true} if this needs to be remembered
     * @hide
     */
    public void setFrequencyBand(int band, boolean persist) {
        try {
            mService.setFrequencyBand(band, persist);
        } catch (RemoteException e) { }
    }
```

android.config//wpa_supplicant宏控制开关,如EAP协议等
