# WIFI 剖析
> 内容来自cnblog http://www.cnblogs.com/hdk1993/p/4771291.html

## wpa_supplicant是什么？
wpa_supplicant 是跨平台的 WPA 请求者程序（supplicant），支持 WEP、WPA 和 WPA2（IEEE 802.11i / RSN （健壮安全网络-Robust Secure Network））。可以在桌面、笔记本甚至嵌入式系统中使用。

## Android中的WIFI

### 介绍
1. WIFI的daemon wpa_supplicant
2. 实现在 /external/wpa_suplicant
3. wpa_supplicant适配层是通用的wpa_supplicant的封装，在Android中作为WIFI部分的硬件抽象层(HAL)来使用。wpa_supplicant适配层主要用于封装与wpa_supplicant守护进程的通信，以提供给Android框架使用。它实现了加载，控制和消息监控等功能。
4. wpa_supplicant适配层的头文件
> /hardware/libhardware_legacy/include/hardware_legacy/wifi.h

### 加载过程
1. 系统启动在init.rc中加载
```
// init.rc
    mkdir /data/misc/wifi/sockets 0770 wifi wifi
    mkdir /data/misc/wifi/wpa_supplicant 0770 wifi wifi
    mkdir /data/misc/ethernet 0770 system system
    mkdir /data/misc/dhcp 0770 dhcp dhcp
    mkdir /data/misc/user 0771 root root
    mkdir /data/misc/perfprofd 0775 root root
    # give system access to wpa_supplicant.conf for backup and restore
    chmod 0660 /data/misc/wifi/wpa_supplicant.conf

    on boot
    # basic network init
    ifup lo
    hostname localhost
    domainname localdomain

// init.qcom.rc

    # Create the directories used by the Wireless subsystem
    mkdir /data/misc/wifi 0770 wifi wifi
    mkdir /data/misc/wifi/sockets 0770 wifi wifi
    mkdir /data/misc/wifi/wpa_supplicant 0770 wifi wifi
    mkdir /data/misc/dhcp 0770 dhcp dhcp
    chown dhcp dhcp /data/misc/dhcp

    #Create the symlink to qcn wpa_supplicant folder for ar6000 wpa_supplicant
    mkdir /data/system 0775 system system
    #symlink /data/misc/wifi/wpa_supplicant /data/system/wpa_supplicant

on property:init.svc.wpa_supplicant=stopped
    stop dhcpcd

service p2p_supplicant /system/bin/wpa_supplicant \
    -ip2p0 -Dnl80211 -c/data/misc/wifi/p2p_supplicant.conf \
    -I/system/etc/wifi/p2p_supplicant_overlay.conf -N \
    -iwlan0 -Dnl80211 -c/data/misc/wifi/wpa_supplicant.conf \
    -I/system/etc/wifi/wpa_supplicant_overlay.conf \
    -O/data/misc/wifi/sockets -puse_p2p_group_interface=1 -dd \
    -e/data/misc/wifi/entropy.bin -g@android:wpa_wlan0
#   we will start as root and wpa_supplicant will switch to user wifi
#   after setting up the capabilities required for WEXT
#   user wifi
#   group wifi inet keystore
    class main
    socket wpa_wlan0 dgram 660 wifi wifi
    disabled
    oneshot

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

# FST Manager can be started by property_set("ctl.start", "fstman:<hostap ctrl iface>");
service fstman /system/bin/fstman -B -ddd -c /data/misc/wifi/fstman.ini
    user wifi
    group wifi net_admin net_raw
    class main
    disabled
    oneshot
```

linux内核模块`/system/lib/modules/wlan.ko` 添加_wpa_supplicant.conf_
> http://w1.fi/gitweb/gitweb.cgi?p=hostap.git;a=blob_plain;f=wpa_supplicant/wpa_supplicant.conf

```
// /etc/wifi/wpa_supplicant.conf
pdate_config=1
eapol_version=1
ap_scan=1
fast_reauth=1
pmf=1
p2p_no_group_iface=1

// /etc/wifi/wpa_supplicant_overlay.conf

disable_scan_offload=1
p2p_disabled=1
```

2. 模块定义和加载`/hardware/libhardware_legacy/wifi/wifi.c` ====> wifi_load_driver() /system/lib/modules/wlan.ko

```c
int wifi_load_driver()
{
#ifdef WIFI_DRIVER_MODULE_PATH
    char driver_status[PROPERTY_VALUE_MAX];
    int count = 100; /* wait at most 20 seconds for completion */

    if (is_wifi_driver_loaded()) {
        return 0;
    }

    if (insmod(DRIVER_MODULE_PATH, DRIVER_MODULE_ARG) < 0)
        return -1;

    if (strcmp(FIRMWARE_LOADER,"") == 0) {
        /* usleep(WIFI_DRIVER_LOADER_DELAY); */
        property_set(DRIVER_PROP_NAME, "ok"); //set ok wifi loaded
    }
    else {
        property_set("ctl.start", FIRMWARE_LOADER);
    }
    sched_yield();
    while (count-- > 0) {
        if (property_get(DRIVER_PROP_NAME, driver_status, NULL)) {
            if (strcmp(driver_status, "ok") == 0)
                return wifi_fst_load_driver();
            else if (strcmp(driver_status, "failed") == 0) {
                wifi_unload_driver();
                return -1;
            }
        }
        usleep(200000);
    }
    property_set(DRIVER_PROP_NAME, "timeout");
    wifi_unload_driver();
    return -1;
#else
#ifdef WIFI_DRIVER_STATE_CTRL_PARAM
    if (is_wifi_driver_loaded()) {
        return 0;
    }

    if (wifi_change_driver_state(WIFI_DRIVER_STATE_ON) < 0)
        return -1;
#endif
    property_set(DRIVER_PROP_NAME, "ok");
    return 0;
#endif


// wifi_fst_load_driver()
int wifi_fst_load_driver()
{
    if (!is_fst_enabled())
        return 0;

    if (is_fst_driver_loaded())
        return 0;

    if (insmod(WIFI_FST_DRIVER_MODULE_PATH, WIFI_FST_DRIVER_MODULE_ARG) < 0)
        return -1;

    property_set(FST_DRIVER_PROP_NAME, "ok");

    return 0;
}
```

3. wifi_start_supplicant
property_set("ctl.start",
