## WIFI
### WIFI open

```
	在WifiController文件中的的状态机 "部分结构" 如下[其中mApStaDisabledState为状态机初始状态]:

		  mDefaultState.....................................................
			   |                                                    |
	   mApStaDisabledState       mStaEnabledState.....................
																	|
												  mDeviceActiveState
																	 |
												  DeviceActiveHighPerfState

	简单表示如下:
										mP0(mDefaultState)
										  /           \
	(mStaEnabledState)mP1    mS0(mApStaDisabledState)
										/   \
											mS1(mDeviceActiveState)


	当前状态为:mP0(mDefaultState),mS0(mApStaDisabledState)
	状态的切换实际是状态机的切换，切换状态机并进入enter()，执行操作，postMessage()接受消息，执行操作
```

1. 在状态机起始状态，接收到CMD_WIFI_TOGGLED信号
/frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiController.java
```java
430    class ApStaDisabledState extends State {
431        private int mDeferredEnableSerialNumber = 0;
432        private boolean mHaveDeferredEnable = false;
433        private long mDisabledTimestamp;
434
435        @Override
436        public void enter() {
437            mWifiStateMachine.setSupplicantRunning(false);
438            // Supplicant can't restart right away, so not the time we switched off
439            mDisabledTimestamp = SystemClock.elapsedRealtime();
440            mDeferredEnableSerialNumber++;
441            mHaveDeferredEnable = false;
442            mWifiStateMachine.clearANQPCache();
443        }
444        @Override
445        public boolean processMessage(Message msg) {
446            switch (msg.what) {
447                case CMD_WIFI_TOGGLED:
448                case CMD_AIRPLANE_TOGGLED:
```

2. 切换状态机前的准备
```java
445        public boolean processMessage(Message msg) {
446            switch (msg.what) {
447                case CMD_WIFI_TOGGLED:
448                case CMD_AIRPLANE_TOGGLED:
449                    if (mSettingsStore.isWifiToggleEnabled()) {
450                        if (doDeferEnable(msg)) {
451                            if (mHaveDeferredEnable) {  //是否需要进行延时操作，放置频繁的开关wifi，默认500
452                                //  have 2 toggles now, inc serial number an ignore both
453                                mDeferredEnableSerialNumber++;
454                            }
455                            mHaveDeferredEnable = !mHaveDeferredEnable;
456                            break;
457                        }
458                        if (mDeviceIdle == false) {
```

3. 切换状态机 
```java
458                        if (mDeviceIdle == false) {
459                            // wifi is toggled, we need to explicitly tell WifiStateMachine that we
460                            // are headed to connect mode before going to the DeviceActiveState
461                            // since that will start supplicant and WifiStateMachine may not know
462                            // what state to head to (it might go to scan mode).
463                            mWifiStateMachine.setOperationalMode(WifiStateMachine.CONNECT_MODE); //a.切换模式
464                            transitionTo(mDeviceActiveState); //b.切换状态机
465                        } else {
466                            checkLocksAndTransitionWhenDeviceIdle();
467                        }
```
3a. 切换模式
```java
// /frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiStateMachine.java

3995    class InitialState extends State {
···
4011        @Override
4012        public boolean processMessage(Message message) {
4013            logStateAndMessage(message, this);
4014            switch (message.what) {
4015                case CMD_START_SUPPLICANT: // jump here
4016                    mClientInterface = mWifiNative.setupForClientMode(); // 3a1-->3a2 open hw 获取scanner & 设置
4017                    if (mClientInterface == null
4018                            || !mDeathRecipient.linkToDeath(mClientInterface.asBinder())) {
4019                        setWifiState(WifiManager.WIFI_STATE_UNKNOWN);
4020                        cleanup();
4021                        break;
4022                    }
4023
4024                    try {
4025                        // A runtime crash or shutting down AP mode can leave
4026                        // IP addresses configured, and this affects
4027                        // connectivity when supplicant starts up.
4028                        // Ensure we have no IP addresses before a supplicant start.
4029                        mNwService.clearInterfaceAddresses(mInterfaceName);
4030
4031                        // Set privacy extensions
4032                        mNwService.setInterfaceIpv6PrivacyExtensions(mInterfaceName, true);
4033
4034                        // IPv6 is enabled only as long as access point is connected since:
4035                        // - IPv6 addresses and routes stick around after disconnection
4036                        // - kernel is unaware when connected and fails to start IPv6 negotiation
4037                        // - kernel can start autoconfiguration when 802.1x is not complete
4038                        mNwService.disableIpv6(mInterfaceName);
4039                    } catch (RemoteException re) {
4040                        loge("Unable to change interface settings: " + re);
4041                    } catch (IllegalStateException ie) {
4042                        loge("Unable to change interface settings: " + ie);
4043                    }
4044
4045                    if (!mWifiNative.enableSupplicant()) { // / 3b1-->3b2???
4046                        loge("Failed to start supplicant!");
4047                        setWifiState(WifiManager.WIFI_STATE_UNKNOWN);
4048                        cleanup();
4049                        break;
4050                    }
4051                    if (mVerboseLoggingEnabled) log("Supplicant start successful");
4052                    mWifiMonitor.startMonitoring(mInterfaceName, true);
4053                    setSupplicantLogLevel();
4054                    transitionTo(mSupplicantStartingState);
4055                    break;
4056                case CMD_START_AP:
4057                    transitionTo(mSoftApState);
4058                    break;
4059                case CMD_SET_OPERATIONAL_MODE: //this
4060                    mOperationalMode = message.arg1;
4061                    if (mOperationalMode != DISABLED_MODE) {
4062                        sendMessage(CMD_START_SUPPLICANT); //jump case CMD_START_SUPPLICANT
4063                    }
4064                    break;
4065                default:
4066                    return NOT_HANDLED;
4067            }
4068            return HANDLED;
```

3a.1. setupForClientMode
```java
90   /********************************************************
91    * Native Initialization/Deinitialization
92    ********************************************************/
93
94   /**
95    * Setup wifi native for Client mode operations.
96    *
97    * 1. Starts the Wifi HAL and configures it in client/STA mode.
98    * 2. Setup Wificond to operate in client mode and retrieve the handle to use for client
99    * operations.
100    *
101    * @return An IClientInterface as wificond client interface binder handler.
102    * Returns null on failure.
103    */
104    public IClientInterface setupForClientMode() {
105        if (!startHalIfNecessary(true)) {
106            Log.e(mTAG, "Failed to start HAL for client mode");
107            return null;
108        }
109        return mWificondControl.setupDriverForClientMode();
110    }
```

3a.2. startHalIfNecessary
```java
// /frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiNative.java
820    /**
821     * Bring up the Vendor HAL and configure for STA mode or AP mode, if vendor HAL is supported.
822     *
823     * @param isStaMode true to start HAL in STA mode, false to start in AP mode.
824     * @return false if the HAL start fails, true if successful or if vendor HAL not supported.
825     */
826    private boolean startHalIfNecessary(boolean isStaMode) {
827        if (!mWifiVendorHal.isVendorHalSupported()) {
828            Log.i(mTAG, "Vendor HAL not supported, Ignore start...");
829            return true;
830        }
831        return mWifiVendorHal.startVendorHal(isStaMode);
832    }
833
```

3a.2.1  startVendorHal
```java
// /frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiVendorHal.java
292    public boolean startVendorHal(boolean isStaMode) {
293        synchronized (sLock) {
294            if (mIWifiStaIface != null) return boolResult(false);
295            if (mIWifiApIface != null) return boolResult(false);
296            if (!mHalDeviceManager.start()) {
/*
// /frameworks/opt/net/wifi/service/java/com/android/server/wifi/HalDeviceManager.java
	137    /**
	138     * Attempts to start Wi-Fi (using HIDL). Returns the success (true) or failure (false) or
	139     * the start operation. Will also dispatch any registered ManagerStatusCallback.onStart() on
	140     * success.
	141     *
	142     * Note: direct call to HIDL.
	143     */
	144    public boolean start() {
	145        return startWifi();
	146    }
	147
	
	1088    private boolean startWifi() {
	1089        if (DBG) Log.d(TAG, "startWifi");
	1090
	1091        synchronized (mLock) {
	1092            try {
	1093                if (mWifi == null) {
	1094                    Log.w(TAG, "startWifi called but mWifi is null!?");
	1095                    return false;
	1096                } else {
	1097                    int triedCount = 0;
	1098                    while (triedCount <= START_HAL_RETRY_TIMES) {
	1099                        WifiStatus status = mWifi.start(); ////important!!!
	1100                        if (status.code == WifiStatusCode.SUCCESS) {
	1101                            initIWifiChipDebugListeners();
	1102                            managerStatusListenerDispatch();
	1103                            if (triedCount != 0) {
	1104                                Log.d(TAG, "start IWifi succeeded after trying "
	1105                                         + triedCount + " times");
	1106                            }
	1107                            return true;
	1108                        } else if (status.code == WifiStatusCode.ERROR_NOT_AVAILABLE) {
	1109                            // Should retry. Hal might still be stopping.
	1110                            Log.e(TAG, "Cannot start IWifi: " + statusString(status)
	1111                                    + ", Retrying...");
	1112                            try {
	1113                                Thread.sleep(START_HAL_RETRY_INTERVAL_MS);
	1114                            } catch (InterruptedException ignore) {
	1115                                // no-op
	1116                            }
	1117                            triedCount++;
	1118                        } else {
	1119                            // Should not retry on other failures.
	1120                            Log.e(TAG, "Cannot start IWifi: " + statusString(status));
	1121                            return false;
	1122                        }
	1123                    }
	1124                    Log.e(TAG, "Cannot start IWifi after trying " + triedCount + " times");
	1125                    return false;
	1126                }
	1127            } catch (RemoteException e) {
	1128                Log.e(TAG, "startWifi exception: " + e);
	1129                return false;
	1130            }
	1131        }
	1132    }
*/
297                return startFailedTo("start the vendor HAL");
298            }
299            IWifiIface iface;
300            if (isStaMode) {
301                mIWifiStaIface = mHalDeviceManager.createStaIface(null, null);
302                if (mIWifiStaIface == null) {
303                    return startFailedTo("create STA Iface");
304                }
305                iface = (IWifiIface) mIWifiStaIface;
306                if (!registerStaIfaceCallback()) {
307                    return startFailedTo("register sta iface callback");
308                }
309                mIWifiRttController = mHalDeviceManager.createRttController(iface);
310                if (mIWifiRttController == null) {
311                    return startFailedTo("create RTT controller");
312                }
313                if (!registerRttEventCallback()) {
314                    return startFailedTo("register RTT iface callback");
315                }
316                enableLinkLayerStats();
317            } else {
318                mIWifiApIface = mHalDeviceManager.createApIface(null, null);
319                if (mIWifiApIface == null) {
320                    return startFailedTo("create AP Iface");
321                }
322                iface = (IWifiIface) mIWifiApIface;
323            }
324            mIWifiChip = mHalDeviceManager.getChip(iface);
325            if (mIWifiChip == null) {
326                return startFailedTo("get the chip created for the Iface");
327            }
328            if (!registerChipCallback()) {
329                return startFailedTo("register chip callback");
330            }
331            mLog.i("Vendor Hal started successfully");
332            return true;
333        }
334    }
```

3a.3. setupDriverForClientMode 
```java
107    /**
108    * Setup driver for client mode via wificond.
109    * @return An IClientInterface as wificond client interface binder handler.
110    * Returns null on failure.
111    */
112    public IClientInterface setupDriverForClientMode() {
113        Log.d(TAG, "Setting up driver for client mode");
114        mWificond = mWifiInjector.makeWificond();
	/*
	/frameworks/opt/net/wifi/service/java/com/android/server/wifi/WificondControl.java
	360    public IWificond makeWificond() {
	361        // We depend on being able to refresh our binder in WifiStateMachine, so don't cache it.
	362        IBinder binder = ServiceManager.getService(WIFICOND_SERVICE_NAME);
	363        return IWificond.Stub.asInterface(binder);
	364    }
	365
	*/
115        if (mWificond == null) {
116            Log.e(TAG, "Failed to get reference to wificond");
117            return null;
118        }
119
120        IClientInterface clientInterface = null;
121        try {
122            clientInterface = mWificond.createClientInterface(); //感觉没干嘛 就是创建了个interface
	/*
	/system/connectivity/wificond/server.cpp

	136Status Server::createClientInterface(sp<IClientInterface>* created_interface) {
	137  InterfaceInfo interface;
	138  if (!SetupInterface(&interface)) {
	139    return Status::ok();  // Logging was done internally
	140  }
	141
	142  unique_ptr<ClientInterfaceImpl> client_interface(new ClientInterfaceImpl(
	143      wiphy_index_,
	144      interface.name,
	145      interface.index,
	146      interface.mac_address,
	147      if_tool_.get(),
	148      supplicant_manager_.get(),
	149      netlink_utils_,
	150      scan_utils_));
	151  *created_interface = client_interface->GetBinder();
	152  client_interfaces_.push_back(std::move(client_interface));
	153  BroadcastClientInterfaceReady(client_interfaces_.back()->GetBinder());
	154
	155  return Status::ok();
	156}
	*/
123        } catch (RemoteException e1) {
124            Log.e(TAG, "Failed to get IClientInterface due to remote exception");
125            return null;
126        }
127
128        if (clientInterface == null) {
129            Log.e(TAG, "Could not get IClientInterface instance from wificond");
130            return null;
131        }
132        Binder.allowBlocking(clientInterface.asBinder());
133
134        // Refresh Handlers
135        mClientInterface = clientInterface;
136        try {
137            mClientInterfaceName = clientInterface.getInterfaceName();
138            mWificondScanner = mClientInterface.getWifiScannerImpl();
139            if (mWificondScanner == null) {
140                Log.e(TAG, "Failed to get WificondScannerImpl");
141                return null;
142            }
143            Binder.allowBlocking(mWificondScanner.asBinder());
144            mScanEventHandler = new ScanEventHandler();
145            mWificondScanner.subscribeScanEvents(mScanEventHandler);
146            mPnoScanEventHandler = new PnoScanEventHandler();
147            mWificondScanner.subscribePnoScanEvents(mPnoScanEventHandler);
148        } catch (RemoteException e) {
149            Log.e(TAG, "Failed to refresh wificond scanner due to remote exception");
150        }
151
152        return clientInterface;
153    }
154
```


3b1. enableSupplicant
```cpp
// /system/connectivity/wificond/client_interface_binder.cpp
40Status ClientInterfaceBinder::enableSupplicant(bool* success) {
41  *success = impl_ && impl_->EnableSupplicant();
	/*
		171bool ClientInterfaceImpl::EnableSupplicant() {
		172  return supplicant_manager_->StartSupplicant();
		/*
		/frameworks/opt/net/wifi/libwifi_system/supplicant_manager.cpp
			128 bool SupplicantManager::StartSupplicant() {
			129  char supp_status[PROPERTY_VALUE_MAX] = {'\0'};
			130  int count = 200; /* wait at most 20 seconds for completion */
			131  const prop_info* pi;
			132  unsigned serial = 0;
			133
			134  /* Check whether already running */
			135  if (property_get(kSupplicantInitProperty, supp_status, NULL) &&
			136      strcmp(supp_status, "running") == 0) {
			137    return true;
			138  }
			139
			140  /* Before starting the daemon, make sure its config file exists */
			141  if (ensure_config_file_exists(kSupplicantConfigFile) < 0) {
			142    LOG(ERROR) << "Wi-Fi will not be enabled";
			143    return false;
			144  }
			145
			146  /*
			147   * Some devices have another configuration file for the p2p interface.
			148   * However, not all devices have this, and we'll let it slide if it
			149   * is missing.  For devices that do expect this file to exist,
			150   * supplicant will refuse to start and emit a good error message.
			151   * No need to check for it here.
			152   */
			153  (void)ensure_config_file_exists(kP2pConfigFile);
			154
			155  if (!EnsureEntropyFileExists()) {
			156    LOG(ERROR) << "Wi-Fi entropy file was not created";
			157  }
			158
			159  /*
			160   * Get a reference to the status property, so we can distinguish
			161   * the case where it goes stopped => running => stopped (i.e.,
			162   * it start up, but fails right away) from the case in which
			163   * it starts in the stopped state and never manages to start
			164   * running at all.
			165   */
			166  pi = __system_property_find(kSupplicantInitProperty);
			167  if (pi != NULL) {
			168    serial = __system_property_serial(pi);
			169  }
			170
			171  property_set("ctl.start", kSupplicantServiceName);
			172  sched_yield();
			173
			174  while (count-- > 0) {
			175    if (pi == NULL) {
			176      pi = __system_property_find(kSupplicantInitProperty);
			177    }
			178    if (pi != NULL) {
			179      /*
			180       * property serial updated means that init process is scheduled
			181       * after we sched_yield, further property status checking is based on this
			182       */
			183      if (__system_property_serial(pi) != serial) {
			184        __system_property_read(pi, NULL, supp_status);
			185        if (strcmp(supp_status, "running") == 0) {
			186          return true;
			187        } else if (strcmp(supp_status, "stopped") == 0) {
			188          return false;
			189        }
			190      }
			191    }
			192    usleep(100000);
			193  }
			194  return false;
			195}
		*/
		173}
	*/
	
42  return Status::ok();
43}
```