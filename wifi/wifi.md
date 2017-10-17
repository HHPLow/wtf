## WIFI
### 一、WIFI open

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
4054                    transitionTo(mSupplicantStartingState); // 结束此状态 转到下一状态
/*
	4072    class SupplicantStartingState extends State {
	4073        private void initializeWpsDetails() {
	4074            String detail;
	4075            detail = mPropertyService.get("ro.product.name", "");
	4076            if (!mWifiNative.setDeviceName(detail)) {
	4077                loge("Failed to set device name " +  detail);
	4078            }
	4079            detail = mPropertyService.get("ro.product.manufacturer", "");
	4080            if (!mWifiNative.setManufacturer(detail)) {
	4081                loge("Failed to set manufacturer " + detail);
	4082            }
	4083            detail = mPropertyService.get("ro.product.model", "");
	4084            if (!mWifiNative.setModelName(detail)) {
	4085                loge("Failed to set model name " + detail);
	4086            }
	4087            detail = mPropertyService.get("ro.product.model", "");
	4088            if (!mWifiNative.setModelNumber(detail)) {
	4089                loge("Failed to set model number " + detail);
	4090            }
	4091            detail = mPropertyService.get("ro.serialno", "");
	4092            if (!mWifiNative.setSerialNumber(detail)) {
	4093                loge("Failed to set serial number " + detail);
	4094            }
	4095            if (!mWifiNative.setConfigMethods("physical_display virtual_push_button")) {
	4096                loge("Failed to set WPS config methods");
	4097            }
	4098            if (!mWifiNative.setDeviceType(mPrimaryDeviceType)) {
	4099                loge("Failed to set primary device type " + mPrimaryDeviceType);
	4100            }
	4101        }
	4102
*/
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
	1099                        WifiStatus status = mWifi.start();                 ////important!!!
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

http://androidxref.com/8.0.0_r4/xref/frameworks/opt/net/wifi/libwifi_system/supplicant_manager.cpp#128
http://androidxref.com/8.0.0_r4/xref/frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiNative.java#mWificondControl
http://androidxref.com/8.0.0_r4/xref/frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiStateMachine.java


### WIFI scan
1. 万物始于WifiManager.java
```java
// /frameworks/base/wifi/java/android/net/wifi/WifiManager.java

1521    /**
1522     * Request a scan for access points. Returns immediately. The availability
1523     * of the results is made known later by means of an asynchronous event sent
1524     * on completion of the scan.
1525     * @return {@code true} if the operation succeeded, i.e., the scan was initiated
1526     */
1527    public boolean startScan() {
1528        return startScan(null);
1529    }
1530
1531    /** @hide */
1532    @SystemApi
1533    @RequiresPermission(android.Manifest.permission.UPDATE_DEVICE_STATS)
1534    public boolean startScan(WorkSource workSource) {
1535        try {
1536            String packageName = mContext.getOpPackageName();
1537            mService.startScan(null, workSource, packageName); 
1538            return true;
1539        } catch (RemoteException e) {
1540            throw e.rethrowFromSystemServer();
1541        }
1542    }
1543
```

2. startScan 
```java
// /frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiServiceImpl.java
538    /**
539     * see {@link android.net.wifi.WifiManager#startScan}
540     * and {@link android.net.wifi.WifiManager#startCustomizedScan}
541     *
542     * @param settings If null, use default parameter, i.e. full scan.
543     * @param workSource If null, all blame is given to the calling uid.
544     * @param packageName Package name of the app that requests wifi scan.
545     */
546    @Override
547    public void startScan(ScanSettings settings, WorkSource workSource, String packageName) {
548        enforceChangePermission();
549
550        mLog.trace("startScan uid=%").c(Binder.getCallingUid()).flush();
551        // Check and throttle background apps for wifi scan.
552        if (isRequestFromBackground(packageName)) {
553            long lastScanMs = mLastScanTimestamps.getOrDefault(packageName, 0L);
554            long elapsedRealtime = mClock.getElapsedSinceBootMillis();
555
556            if (lastScanMs != 0 && (elapsedRealtime - lastScanMs) < mBackgroundThrottleInterval) {
557                sendFailedScanBroadcast();
558                return;
559            }
560            // Proceed with the scan request and record the time.
561            mLastScanTimestamps.put(packageName, elapsedRealtime);
562        }
563        synchronized (this) {
564            if (mWifiScanner == null) {
565                mWifiScanner = mWifiInjector.getWifiScanner();
566            }
567            if (mInIdleMode) {
568                // Need to send an immediate scan result broadcast in case the
569                // caller is waiting for a result ..
570
571                // TODO: investigate if the logic to cancel scans when idle can move to
572                // WifiScanningServiceImpl.  This will 1 - clean up WifiServiceImpl and 2 -
573                // avoid plumbing an awkward path to report a cancelled/failed scan.  This will
574                // be sent directly until b/31398592 is fixed.
575                sendFailedScanBroadcast();
576                mScanPending = true;
577                return;
578            }
579        }
580        if (settings != null) {
581            settings = new ScanSettings(settings);
582            if (!settings.isValid()) {
583                Slog.e(TAG, "invalid scan setting");
584                return;
585            }
586        }
587        if (workSource != null) {
588            enforceWorkSourcePermission();
589            // WifiManager currently doesn't use names, so need to clear names out of the
590            // supplied WorkSource to allow future WorkSource combining.
591            workSource.clearNames();
592        }
593        if (workSource == null && Binder.getCallingUid() >= 0) {
594            workSource = new WorkSource(Binder.getCallingUid());
595        }
596        mWifiStateMachine.startScan(Binder.getCallingUid(), scanRequestCounter++,
597                settings, workSource);
598    }
```

3. 又回到了状态机 startScan
```java
// /frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiStateMachine.java
1326    /**
1327     * Initiate a wifi scan. If workSource is not null, blame is given to it, otherwise blame is
1328     * given to callingUid.
1329     *
1330     * @param callingUid The uid initiating the wifi scan. Blame will be given here unless
1331     *                   workSource is specified.
1332     * @param workSource If not null, blame is given to workSource.
1333     * @param settings   Scan settings, see {@link ScanSettings}.
1334     */
1335    public void startScan(int callingUid, int scanCounter,
1336                          ScanSettings settings, WorkSource workSource) {
1337        Bundle bundle = new Bundle();
1338        bundle.putParcelable(CUSTOMIZED_SCAN_SETTING, settings);
1339        bundle.putParcelable(CUSTOMIZED_SCAN_WORKSOURCE, workSource);
1340        bundle.putLong(SCAN_REQUEST_TIME, mClock.getWallClockMillis());
1341        sendMessage(CMD_START_SCAN, callingUid, scanCounter, bundle);
1342    }
```

4. 开启wifi之后才能扫描 所以在SupplicantStartedState
```java
4159    class SupplicantStartedState extends State {
4160        @Override
4161        public void enter() {
4162            if (mVerboseLoggingEnabled) {
4163                logd("SupplicantStartedState enter");
4164            }
4165
4166            mWifiNative.setExternalSim(true);
4167
4168            setRandomMacOui();
4169            mCountryCode.setReadyForChange(true);
4170
4171            // We can't do this in the constructor because WifiStateMachine is created before the
4172            // wifi scanning service is initialized
4173            if (mWifiScanner == null) {
4174                mWifiScanner = mWifiInjector.getWifiScanner();
4175
4176                synchronized (mWifiReqCountLock) {
4177                    mWifiConnectivityManager =
4178                            mWifiInjector.makeWifiConnectivityManager(mWifiInfo,
4179                                                                      hasConnectionRequests());
4180                    mWifiConnectivityManager.setUntrustedConnectionAllowed(mUntrustedReqCount > 0);
4181                    mWifiConnectivityManager.handleScreenStateChanged(mScreenOn);
4182                }
4183            }
4184
4185            mWifiDiagnostics.startLogging(mVerboseLoggingEnabled);
4186            mIsRunning = true;
4187            updateBatteryWorkSource(null);
4188            /**
4189             * Enable bluetooth coexistence scan mode when bluetooth connection is active.
4190             * When this mode is on, some of the low-level scan parameters used by the
4191             * driver are changed to reduce interference with bluetooth
4192             */
4193            mWifiNative.setBluetoothCoexistenceScanMode(mBluetoothConnectionActive);
4194            // initialize network state
4195            setNetworkDetailedState(DetailedState.DISCONNECTED);
4196
4197            // Disable legacy multicast filtering, which on some chipsets defaults to enabled.
4198            // Legacy IPv6 multicast filtering blocks ICMPv6 router advertisements which breaks IPv6
4199            // provisioning. Legacy IPv4 multicast filtering may be re-enabled later via
4200            // IpManager.Callback.setFallbackMulticastFilter()
4201            mWifiNative.stopFilteringMulticastV4Packets();
4202            mWifiNative.stopFilteringMulticastV6Packets();
4203
4204            if (mOperationalMode == SCAN_ONLY_MODE ||
4205                    mOperationalMode == SCAN_ONLY_WITH_WIFI_OFF_MODE) {
4206                mWifiNative.disconnect();
4207                setWifiState(WIFI_STATE_DISABLED);
4208                transitionTo(mScanModeState);
4209            } else if (mOperationalMode == CONNECT_MODE) {
4210                setWifiState(WIFI_STATE_ENABLING);
4211                // Transitioning to Disconnected state will trigger a scan and subsequently AutoJoin
4212                transitionTo(mDisconnectedState);
4213            } else if (mOperationalMode == DISABLED_MODE) {
4214                transitionTo(mSupplicantStoppingState);
4215            }
4216
4217            // Set the right suspend mode settings
4218            mWifiNative.setSuspendOptimizations(mSuspendOptNeedsDisabled == 0
4219                    && mUserWantsSuspendOpt.get());
4220
4221            mWifiNative.setPowerSave(true);
4222
4223            if (mP2pSupported) {
4224                if (mOperationalMode == CONNECT_MODE) {
4225                    p2pSendMessage(WifiStateMachine.CMD_ENABLE_P2P);
4226                } else {
4227                    // P2P state machine starts in disabled state, and is not enabled until
4228                    // CMD_ENABLE_P2P is sent from here; so, nothing needs to be done to
4229                    // keep it disabled.
4230                }
4231            }
4232
4233            final Intent intent = new Intent(WifiManager.WIFI_SCAN_AVAILABLE);
4234            intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT);
4235            intent.putExtra(WifiManager.EXTRA_SCAN_AVAILABLE, WIFI_STATE_ENABLED);
4236            mContext.sendStickyBroadcastAsUser(intent, UserHandle.ALL);
4237
4238            // Disable wpa_supplicant from auto reconnecting.
4239            mWifiNative.enableStaAutoReconnect(false);
4240            // STA has higher priority over P2P
4241            mWifiNative.setConcurrencyPriority(true);
4242        }
4243
4244        @Override
4245        public boolean processMessage(Message message) {
4246            logStateAndMessage(message, this);
4247
4248            switch(message.what) {
4249                case CMD_STOP_SUPPLICANT:   /* Supplicant stopped by user */
4250                    if (mP2pSupported) {
4251                        transitionTo(mWaitForP2pDisableState);
4252                    } else {
4253                        transitionTo(mSupplicantStoppingState);
4254                    }
4255                    break;
4256                case WifiMonitor.SUP_DISCONNECTION_EVENT:  /* Supplicant connection lost */
4257                    loge("Connection lost, restart supplicant");
4258                    handleSupplicantConnectionLoss(true);
4259                    handleNetworkDisconnect();
4260                    mSupplicantStateTracker.sendMessage(CMD_RESET_SUPPLICANT_STATE);
4261                    if (mP2pSupported) {
4262                        transitionTo(mWaitForP2pDisableState);
4263                    } else {
4264                        transitionTo(mInitialState);
4265                    }
4266                    sendMessageDelayed(CMD_START_SUPPLICANT, SUPPLICANT_RESTART_INTERVAL_MSECS);
4267                    break;
4268                case CMD_START_SCAN:
4269                    // TODO: remove scan request path (b/31445200)
4270                    handleScanRequest(message);
4271                    break;
4272                case WifiMonitor.SCAN_RESULTS_EVENT:
4273                case WifiMonitor.SCAN_FAILED_EVENT:
4274                    // TODO: remove handing of SCAN_RESULTS_EVENT and SCAN_FAILED_EVENT when scan
4275                    // results are retrieved from WifiScanner (b/31444878)
4276                    maybeRegisterNetworkFactory(); // Make sure our NetworkFactory is registered
4277                    setScanResults();
4278                    mIsScanOngoing = false;
4279                    mIsFullScanOngoing = false;
4280                    if (mBufferedScanMsg.size() > 0)
4281                        sendMessage(mBufferedScanMsg.remove());
4282                    break;
4283                case CMD_START_AP:
4284                    /* Cannot start soft AP while in client mode */
4285                    loge("Failed to start soft AP with a running supplicant");
4286                    setWifiApState(WIFI_AP_STATE_FAILED, WifiManager.SAP_START_FAILURE_GENERAL,
4287                            null, WifiManager.IFACE_IP_MODE_UNSPECIFIED);
4288                    break;
4289                case CMD_SET_OPERATIONAL_MODE:
4290                    mOperationalMode = message.arg1;
4291                    if (mOperationalMode == DISABLED_MODE) {
4292                        transitionTo(mSupplicantStoppingState);
4293                    }
4294                    break;
4295                case CMD_TARGET_BSSID:
4296                    // Trying to associate to this BSSID
4297                    if (message.obj != null) {
4298                        mTargetRoamBSSID = (String) message.obj;
4299                    }
4300                    break;
4301                case CMD_GET_LINK_LAYER_STATS:
4302                    WifiLinkLayerStats stats = getWifiLinkLayerStats();
4303                    replyToMessage(message, message.what, stats);
4304                    break;
4305                case CMD_RESET_SIM_NETWORKS:
4306                    log("resetting EAP-SIM/AKA/AKA' networks since SIM was changed");
4307                    mWifiConfigManager.resetSimNetworks();
4308                    break;
4309                case CMD_BLUETOOTH_ADAPTER_STATE_CHANGE:
4310                    mBluetoothConnectionActive = (message.arg1 !=
4311                            BluetoothAdapter.STATE_DISCONNECTED);
4312                    mWifiNative.setBluetoothCoexistenceScanMode(mBluetoothConnectionActive);
4313                    break;
4314                case CMD_SET_SUSPEND_OPT_ENABLED:
4315                    if (message.arg1 == 1) {
4316                        setSuspendOptimizationsNative(SUSPEND_DUE_TO_SCREEN, true);
4317                        if (message.arg2 == 1) {
4318                            mSuspendWakeLock.release();
4319                        }
4320                    } else {
4321                        setSuspendOptimizationsNative(SUSPEND_DUE_TO_SCREEN, false);
4322                    }
4323                    break;
4324                case CMD_SET_HIGH_PERF_MODE:
4325                    if (message.arg1 == 1) {
4326                        setSuspendOptimizationsNative(SUSPEND_DUE_TO_HIGH_PERF, false);
4327                    } else {
4328                        setSuspendOptimizationsNative(SUSPEND_DUE_TO_HIGH_PERF, true);
4329                    }
4330                    break;
4331                case CMD_ENABLE_TDLS:
4332                    if (message.obj != null) {
4333                        String remoteAddress = (String) message.obj;
4334                        boolean enable = (message.arg1 == 1);
4335                        mWifiNative.startTdls(remoteAddress, enable);
4336                    }
4337                    break;
4338                case WifiMonitor.ANQP_DONE_EVENT:
4339                    // TODO(zqiu): remove this when switch over to wificond for ANQP requests.
4340                    mPasspointManager.notifyANQPDone((AnqpEvent) message.obj);
4341                    break;
4342                case CMD_STOP_IP_PACKET_OFFLOAD: {
4343                    int slot = message.arg1;
4344                    int ret = stopWifiIPPacketOffload(slot);
4345                    if (mNetworkAgent != null) {
4346                        mNetworkAgent.onPacketKeepaliveEvent(slot, ret);
4347                    }
4348                    break;
4349                }
4350                case WifiMonitor.RX_HS20_ANQP_ICON_EVENT:
4351                    // TODO(zqiu): remove this when switch over to wificond for icon requests.
4352                    mPasspointManager.notifyIconDone((IconEvent) message.obj);
4353                    break;
4354                case WifiMonitor.HS20_REMEDIATION_EVENT:
4355                    // TODO(zqiu): remove this when switch over to wificond for WNM frames
4356                    // monitoring.
4357                    mPasspointManager.receivedWnmFrame((WnmData) message.obj);
4358                    break;
4359                case CMD_CONFIG_ND_OFFLOAD:
4360                    final boolean enabled = (message.arg1 > 0);
4361                    mWifiNative.configureNeighborDiscoveryOffload(enabled);
4362                    break;
4363                case CMD_ENABLE_WIFI_CONNECTIVITY_MANAGER:
4364                    mWifiConnectivityManager.enable(message.arg1 == 1 ? true : false);
4365                    break;
4366                case CMD_ENABLE_AUTOJOIN_WHEN_ASSOCIATED:
4367                    final boolean allowed = (message.arg1 > 0);
4368                    boolean old_state = mEnableAutoJoinWhenAssociated;
4369                    mEnableAutoJoinWhenAssociated = allowed;
4370                    if (!old_state && allowed && mScreenOn
4371                            && getCurrentState() == mConnectedState) {
4372                        mWifiConnectivityManager.forceConnectivityScan();
4373                    }
4374                    break;
4375                default:
4376                    return NOT_HANDLED;
4377            }
4378            return HANDLED;
4379        }

```

5. handleScanRequest
```java
1474    private void handleScanRequest(Message message) {
1475        ScanSettings settings = null;
1476        WorkSource workSource = null;
1477
1478        // unbundle parameters
1479        Bundle bundle = (Bundle) message.obj;
1480
1481        if (bundle != null) {
1482            settings = bundle.getParcelable(CUSTOMIZED_SCAN_SETTING);
1483            workSource = bundle.getParcelable(CUSTOMIZED_SCAN_WORKSOURCE);
1484        }
1485
1486        Set<Integer> freqs = null;
1487        if (settings != null && settings.channelSet != null) {
1488            freqs = new HashSet<>();
1489            for (WifiChannel channel : settings.channelSet) {
1490                freqs.add(channel.freqMHz);
1491            }
1492        }
1493
1494        // Retrieve the list of hidden network SSIDs to scan for.
1495        List<WifiScanner.ScanSettings.HiddenNetwork> hiddenNetworks =
1496                mWifiConfigManager.retrieveHiddenNetworkList();
1497
#if 0
	2288    /**
	2289     * Retrieves a list of all the saved hidden networks for scans.
	2290     *
	2291     * Hidden network list sent to the firmware has limited size. If there are a lot of saved
	2292     * networks, this list will be truncated and we might end up not sending the networks
	2293     * with the highest chance of connecting to the firmware.
	2294     * So, re-sort the network list based on the frequency of connection to those networks
	2295     * and whether it was last seen in the scan results.
	2296     *
	2297     * @return list of networks with updated priorities.
	2298     */
	2299    public List<WifiScanner.ScanSettings.HiddenNetwork> retrieveHiddenNetworkList() {
	2300        List<WifiScanner.ScanSettings.HiddenNetwork> hiddenList = new ArrayList<>();
	2301        List<WifiConfiguration> networks = new ArrayList<>(getInternalConfiguredNetworks());
	2302        // Remove any permanently disabled networks or non hidden networks.
	2303        Iterator<WifiConfiguration> iter = networks.iterator();
	2304        while (iter.hasNext()) {
	2305            WifiConfiguration config = iter.next();
	2306            if (!config.hiddenSSID ||
	2307                    config.getNetworkSelectionStatus().isNetworkPermanentlyDisabled()) {
	2308                iter.remove();
	2309            }
	2310        }
	2311        Collections.sort(networks, sScanListComparator);
	2312        // Let's use the network list size - 1 as the highest priority and then go down from there.
	2313        // So, the most frequently connected network has the highest priority now.
	2314        int priority = networks.size() - 1;
	2315        for (WifiConfiguration config : networks) {
	2316            hiddenList.add(
	2317                    new WifiScanner.ScanSettings.HiddenNetwork(config.SSID));
	2318            priority--;
	2319        }
	2320        return hiddenList;
	2321    }
	2322
#endif
1498        // call wifi native to start the scan
1499        if (startScanNative(freqs, hiddenNetworks, workSource)) {
1500            // a full scan covers everything, clearing scan request buffer
1501            if (freqs == null)
1502                mBufferedScanMsg.clear();
1503            messageHandlingStatus = MESSAGE_HANDLING_STATUS_OK;
1504            return;
1505        }
1506
1507        // if reach here, scan request is rejected
1508
1509        if (!mIsScanOngoing) {
1510            // if rejection is NOT due to ongoing scan (e.g. bad scan parameters),
1511
1512            // discard this request and pop up the next one
1513            if (mBufferedScanMsg.size() > 0) {
1514                sendMessage(mBufferedScanMsg.remove());
1515            }
1516            messageHandlingStatus = MESSAGE_HANDLING_STATUS_DISCARD;
1517        } else if (!mIsFullScanOngoing) {
1518            // if rejection is due to an ongoing scan, and the ongoing one is NOT a full scan,
1519            // buffer the scan request to make sure specified channels will be scanned eventually
1520            if (freqs == null)
1521                mBufferedScanMsg.clear();
1522            if (mBufferedScanMsg.size() < SCAN_REQUEST_BUFFER_MAX_SIZE) {
1523                Message msg = obtainMessage(CMD_START_SCAN,
1524                        message.arg1, message.arg2, bundle);
1525                mBufferedScanMsg.add(msg);
1526            } else {
1527                // if too many requests in buffer, combine them into a single full scan
1528                bundle = new Bundle();
1529                bundle.putParcelable(CUSTOMIZED_SCAN_SETTING, null);
1530                bundle.putParcelable(CUSTOMIZED_SCAN_WORKSOURCE, workSource);
1531                Message msg = obtainMessage(CMD_START_SCAN, message.arg1, message.arg2, bundle);
1532                mBufferedScanMsg.clear();
1533                mBufferedScanMsg.add(msg);
1534            }
1535            messageHandlingStatus = MESSAGE_HANDLING_STATUS_LOOPED;
1536        } else {
1537            // mIsScanOngoing and mIsFullScanOngoing
1538            messageHandlingStatus = MESSAGE_HANDLING_STATUS_FAIL;
1539        }
1540    }
1541
```

6. startScanNative
```java
1543    // TODO this is a temporary measure to bridge between WifiScanner and WifiStateMachine until
1544    // scan functionality is refactored out of WifiStateMachine.
1545    /**
1546     * return true iff scan request is accepted
1547     */
1548    private boolean startScanNative(final Set<Integer> freqs,
1549            List<WifiScanner.ScanSettings.HiddenNetwork> hiddenNetworkList,
1550            WorkSource workSource) {
1551        WifiScanner.ScanSettings settings = new WifiScanner.ScanSettings();
1552        if (freqs == null) {
1553            settings.band = WifiScanner.WIFI_BAND_BOTH_WITH_DFS;
1554        } else {
1555            settings.band = WifiScanner.WIFI_BAND_UNSPECIFIED;
1556            int index = 0;
1557            settings.channels = new WifiScanner.ChannelSpec[freqs.size()];
1558            for (Integer freq : freqs) {
1559                settings.channels[index++] = new WifiScanner.ChannelSpec(freq);
1560            }
1561        }
1562        settings.reportEvents = WifiScanner.REPORT_EVENT_AFTER_EACH_SCAN
1563                | WifiScanner.REPORT_EVENT_FULL_SCAN_RESULT;
1564
1565        settings.hiddenNetworks =
1566                hiddenNetworkList.toArray(
1567                        new WifiScanner.ScanSettings.HiddenNetwork[hiddenNetworkList.size()]);
1568
1569        WifiScanner.ScanListener nativeScanListener = new WifiScanner.ScanListener() {
1570                // ignore all events since WifiStateMachine is registered for the supplicant events
1571                @Override
1572                public void onSuccess() {
1573                }
1574                @Override
1575                public void onFailure(int reason, String description) {
1576                    mIsScanOngoing = false;
1577                    mIsFullScanOngoing = false;
1578                }
1579                @Override
1580                public void onResults(WifiScanner.ScanData[] results) {
1581                }
1582                @Override
1583                public void onFullResult(ScanResult fullScanResult) {
1584                }
1585                @Override
1586                public void onPeriodChanged(int periodInMs) {
1587                }
1588            };
1589        mWifiScanner.startScan(settings, nativeScanListener, workSource);
1590        mIsScanOngoing = true;
1591        mIsFullScanOngoing = (freqs == null);
1592        lastScanFreqs = freqs;
1593        return true;
1594    }
```

7. startScan
```java
799    /**
800     * starts a single scan and reports results asynchronously
801     * @param settings specifies various parameters for the scan; for more information look at
802     * {@link ScanSettings}
803     * @param workSource WorkSource to blame for power usage
804     * @param listener specifies the object to report events to. This object is also treated as a
805     *                 key for this scan, and must also be specified to cancel the scan. Multiple
806     *                 scans should also not share this object.
807     */
808    @RequiresPermission(android.Manifest.permission.LOCATION_HARDWARE)
809    public void startScan(ScanSettings settings, ScanListener listener, WorkSource workSource) {
810        Preconditions.checkNotNull(listener, "listener cannot be null");
811        int key = addListener(listener);
812        if (key == INVALID_KEY) return;
813        validateChannel();
814        Bundle scanParams = new Bundle();
815        scanParams.putParcelable(SCAN_PARAMS_SCAN_SETTINGS_KEY, settings);
816        scanParams.putParcelable(SCAN_PARAMS_WORK_SOURCE_KEY, workSource);
817        mAsyncChannel.sendMessage(CMD_START_SINGLE_SCAN, 0, key, scanParams); //又回到状态机
818    }
```
8. CMD_START_SINGLE_SCAN
```java
// /frameworks/opt/net/wifi/service/java/com/android/server/wifi/scanner/WifiScanningServiceImpl.java
574        /**
575         * State representing when the driver is running. This state is not meant to be transitioned
576         * directly, but is instead indented as a parent state of ScanningState and IdleState
577         * to hold common functionality and handle cleaning up scans when the driver is shut down.
578         */
579        class DriverStartedState extends State {
580            @Override
581            public void exit() {
582                // clear scan results when scan mode is not active
583                mCachedScanResults.clear();
584
585                mWifiMetrics.incrementScanReturnEntry(
586                        WifiMetricsProto.WifiLog.SCAN_FAILURE_INTERRUPTED,
587                        mPendingScans.size());
588                sendOpFailedToAllAndClear(mPendingScans, WifiScanner.REASON_UNSPECIFIED,
589                        "Scan was interrupted");
590            }
591
592            @Override
593            public boolean processMessage(Message msg) {
594                ClientInfo ci = mClients.get(msg.replyTo);
595
596                switch (msg.what) {
597                    case WifiScanner.CMD_START_SINGLE_SCAN:
598                        mWifiMetrics.incrementOneshotScanCount();
599                        int handler = msg.arg2;
600                        Bundle scanParams = (Bundle) msg.obj;
601                        if (scanParams == null) {
602                            logCallback("singleScanInvalidRequest",  ci, handler, "null params");
603                            replyFailed(msg, WifiScanner.REASON_INVALID_REQUEST, "params null");
604                            return HANDLED;
605                        }
606                        scanParams.setDefusable(true);
607                        ScanSettings scanSettings =
608                                scanParams.getParcelable(WifiScanner.SCAN_PARAMS_SCAN_SETTINGS_KEY);
609                        WorkSource workSource =
610                                scanParams.getParcelable(WifiScanner.SCAN_PARAMS_WORK_SOURCE_KEY);
611                        if (validateScanRequest(ci, handler, scanSettings, workSource)) {
612                            logScanRequest("addSingleScanRequest", ci, handler, workSource,
613                                    scanSettings, null);
614                            replySucceeded(msg);
615
616                            // If there is an active scan that will fulfill the scan request then
617                            // mark this request as an active scan, otherwise mark it pending.
618                            // If were not currently scanning then try to start a scan. Otherwise
619                            // this scan will be scheduled when transitioning back to IdleState
620                            // after finishing the current scan.
621                            if (getCurrentState() == mScanningState) {
622                                if (activeScanSatisfies(scanSettings)) {
623                                    mActiveScans.addRequest(ci, handler, workSource, scanSettings);
624                                } else {
625                                    mPendingScans.addRequest(ci, handler, workSource, scanSettings);
626                                }
627                            } else {
628                                mPendingScans.addRequest(ci, handler, workSource, scanSettings);
629                                tryToStartNewScan();
/*
// /frameworks/opt/net/wifi/service/java/com/android/server/wifi/scanner/WifiScanningServiceImpl.java
		790        void tryToStartNewScan() {
		791            if (mPendingScans.size() == 0) { // no pending requests
		792                return;
		793            }
		794            mChannelHelper.updateChannels();
		795            // TODO move merging logic to a scheduler
		796            WifiNative.ScanSettings settings = new WifiNative.ScanSettings();
		797            settings.num_buckets = 1;
		798            WifiNative.BucketSettings bucketSettings = new WifiNative.BucketSettings();
		799            bucketSettings.bucket = 0;
		800            bucketSettings.period_ms = 0;
		801            bucketSettings.report_events = WifiScanner.REPORT_EVENT_AFTER_EACH_SCAN;
		802
		803            ChannelCollection channels = mChannelHelper.createChannelCollection();
		804            List<WifiNative.HiddenNetwork> hiddenNetworkList = new ArrayList<>();
		805            for (RequestInfo<ScanSettings> entry : mPendingScans) {
		806                channels.addChannels(entry.settings);
		807                if (entry.settings.hiddenNetworks != null) {
		808                    for (int i = 0; i < entry.settings.hiddenNetworks.length; i++) {
		809                        WifiNative.HiddenNetwork hiddenNetwork = new WifiNative.HiddenNetwork();
		810                        hiddenNetwork.ssid = entry.settings.hiddenNetworks[i].ssid;
		811                        hiddenNetworkList.add(hiddenNetwork);
		812                    }
		813                }
		814                if ((entry.settings.reportEvents & WifiScanner.REPORT_EVENT_FULL_SCAN_RESULT)
		815                        != 0) {
		816                    bucketSettings.report_events |= WifiScanner.REPORT_EVENT_FULL_SCAN_RESULT;
		817                }
		818            }
		819            if (hiddenNetworkList.size() > 0) {
		820                settings.hiddenNetworks = new WifiNative.HiddenNetwork[hiddenNetworkList.size()];
		821                int numHiddenNetworks = 0;
		822                for (WifiNative.HiddenNetwork hiddenNetwork : hiddenNetworkList) {
		823                    settings.hiddenNetworks[numHiddenNetworks++] = hiddenNetwork;
		824                }
		825            }
		826
		827            channels.fillBucketSettings(bucketSettings, Integer.MAX_VALUE);
		828
		829            settings.buckets = new WifiNative.BucketSettings[] {bucketSettings};
		830            if (mScannerImpl.startSingleScan(settings, this)) {
		831                // store the active scan settings
		832                mActiveScanSettings = settings;
		833                // swap pending and active scan requests
		834                RequestList<ScanSettings> tmp = mActiveScans;
		835                mActiveScans = mPendingScans;
		836                mPendingScans = tmp;
		837                // make sure that the pending list is clear
		838                mPendingScans.clear();
		839                transitionTo(mScanningState);
		840            } else {
		841                mWifiMetrics.incrementScanReturnEntry(
		842                        WifiMetricsProto.WifiLog.SCAN_UNKNOWN, mPendingScans.size());
		843                // notify and cancel failed scans
		844                sendOpFailedToAllAndClear(mPendingScans, WifiScanner.REASON_UNSPECIFIED,
		845                        "Failed to start single scan");
		846            }
		847        }
*/
630                            }
631                        } else {
632                            logCallback("singleScanInvalidRequest",  ci, handler, "bad request");
633                            replyFailed(msg, WifiScanner.REASON_INVALID_REQUEST, "bad request");
634                            mWifiMetrics.incrementScanReturnEntry(
635                                    WifiMetricsProto.WifiLog.SCAN_FAILURE_INVALID_CONFIGURATION, 1);
636                        }
637                        return HANDLED;
638                    case WifiScanner.CMD_STOP_SINGLE_SCAN:
639                        removeSingleScanRequest(ci, msg.arg2);
640                        return HANDLED;
641                    default:
642                        return NOT_HANDLED;
643                }
644            }
645        }
```


9. startSingleScan
```java
// /frameworks/opt/net/wifi/service/java/com/android/server/wifi/scanner/WificondScannerImpl.java
182    public boolean startSingleScan(WifiNative.ScanSettings settings,
183            WifiNative.ScanEventHandler eventHandler) {
184        if (eventHandler == null || settings == null) {
185            Log.w(TAG, "Invalid arguments for startSingleScan: settings=" + settings
186                    + ",eventHandler=" + eventHandler);
187            return false;
188        }
189        if (mPendingSingleScanSettings != null
190                || (mLastScanSettings != null && mLastScanSettings.singleScanActive)) {
191            Log.w(TAG, "A single scan is already running");
192            return false;
193        }
194        synchronized (mSettingsLock) {
195            mPendingSingleScanSettings = settings;
196            mPendingSingleScanEventHandler = eventHandler;
197            processPendingScans();
198            return true;
199        }
200    }
```

10. processPendingScans
```java
// /frameworks/opt/net/wifi/service/java/com/android/server/wifi/scanner/WificondScannerImpl.java
328    private void processPendingScans() {
329        synchronized (mSettingsLock) {
330            // Wait for the active scan result to come back to reschedule other scans,
331            // unless if HW pno scan is running. Hw PNO scans are paused it if there
332            // are other pending scans,
333            if (mLastScanSettings != null && !mLastScanSettings.hwPnoScanActive) {
334                return;
335            }
336
337            ChannelCollection allFreqs = mChannelHelper.createChannelCollection();
338            Set<String> hiddenNetworkSSIDSet = new HashSet<>();
339            final LastScanSettings newScanSettings =
340                    new LastScanSettings(mClock.getElapsedSinceBootMillis());
341
342            // Update scan settings if there is a pending scan
343            if (!mBackgroundScanPaused) {
344                if (mPendingBackgroundScanSettings != null) {
345                    mBackgroundScanSettings = mPendingBackgroundScanSettings;
346                    mBackgroundScanEventHandler = mPendingBackgroundScanEventHandler;
347                    mNextBackgroundScanPeriod = 0;
348                    mPendingBackgroundScanSettings = null;
349                    mPendingBackgroundScanEventHandler = null;
350                    mBackgroundScanPeriodPending = true;
351                }
352                if (mBackgroundScanPeriodPending && mBackgroundScanSettings != null) {
353                    int reportEvents = WifiScanner.REPORT_EVENT_NO_BATCH; // default to no batch
354                    for (int bucket_id = 0; bucket_id < mBackgroundScanSettings.num_buckets;
355                            ++bucket_id) {
356                        WifiNative.BucketSettings bucket =
357                                mBackgroundScanSettings.buckets[bucket_id];
358                        if (mNextBackgroundScanPeriod % (bucket.period_ms
359                                        / mBackgroundScanSettings.base_period_ms) == 0) {
360                            if ((bucket.report_events
361                                            & WifiScanner.REPORT_EVENT_AFTER_EACH_SCAN) != 0) {
362                                reportEvents |= WifiScanner.REPORT_EVENT_AFTER_EACH_SCAN;
363                            }
364                            if ((bucket.report_events
365                                            & WifiScanner.REPORT_EVENT_FULL_SCAN_RESULT) != 0) {
366                                reportEvents |= WifiScanner.REPORT_EVENT_FULL_SCAN_RESULT;
367                            }
368                            // only no batch if all buckets specify it
369                            if ((bucket.report_events
370                                            & WifiScanner.REPORT_EVENT_NO_BATCH) == 0) {
371                                reportEvents &= ~WifiScanner.REPORT_EVENT_NO_BATCH;
372                            }
373
374                            allFreqs.addChannels(bucket);
375                        }
376                    }
377                    if (!allFreqs.isEmpty()) {
378                        newScanSettings.setBackgroundScan(mNextBackgroundScanId++,
379                                mBackgroundScanSettings.max_ap_per_scan, reportEvents,
380                                mBackgroundScanSettings.report_threshold_num_scans,
381                                mBackgroundScanSettings.report_threshold_percent);
382                    }
383                    mNextBackgroundScanPeriod++;
384                    mBackgroundScanPeriodPending = false;
385                    mAlarmManager.set(AlarmManager.ELAPSED_REALTIME_WAKEUP,
386                            mClock.getElapsedSinceBootMillis()
387                                    + mBackgroundScanSettings.base_period_ms,
388                            BACKGROUND_PERIOD_ALARM_TAG, mScanPeriodListener, mEventHandler);
389                }
390            }
391
392            if (mPendingSingleScanSettings != null) {
393                boolean reportFullResults = false;
394                ChannelCollection singleScanFreqs = mChannelHelper.createChannelCollection();
395                for (int i = 0; i < mPendingSingleScanSettings.num_buckets; ++i) {
396                    WifiNative.BucketSettings bucketSettings =
397                            mPendingSingleScanSettings.buckets[i];
398                    if ((bucketSettings.report_events
399                                    & WifiScanner.REPORT_EVENT_FULL_SCAN_RESULT) != 0) {
400                        reportFullResults = true;
401                    }
402                    singleScanFreqs.addChannels(bucketSettings);
403                    allFreqs.addChannels(bucketSettings);
404                }
405                newScanSettings.setSingleScan(reportFullResults, singleScanFreqs,
406                        mPendingSingleScanEventHandler);
407
408                WifiNative.HiddenNetwork[] hiddenNetworks =
409                        mPendingSingleScanSettings.hiddenNetworks;
410                if (hiddenNetworks != null) {
411                    int numHiddenNetworks =
412                            Math.min(hiddenNetworks.length, MAX_HIDDEN_NETWORK_IDS_PER_SCAN);
413                    for (int i = 0; i < numHiddenNetworks; i++) {
414                        hiddenNetworkSSIDSet.add(hiddenNetworks[i].ssid);
415                    }
416                }
417
418                mPendingSingleScanSettings = null;
419                mPendingSingleScanEventHandler = null;
420            }
421
422            if ((newScanSettings.backgroundScanActive || newScanSettings.singleScanActive)
423                    && !allFreqs.isEmpty()) {
424                pauseHwPnoScan();
425                Set<Integer> freqs = allFreqs.getScanFreqs();
426                boolean success = mWifiNative.scan(freqs, hiddenNetworkSSIDSet); // here 
427                if (success) {
428                    // TODO handle scan timeout
429                    if (DBG) {
430                        Log.d(TAG, "Starting wifi scan for freqs=" + freqs
431                                + ", background=" + newScanSettings.backgroundScanActive
432                                + ", single=" + newScanSettings.singleScanActive);
433                    }
434                    mLastScanSettings = newScanSettings;
435                    mAlarmManager.set(AlarmManager.ELAPSED_REALTIME_WAKEUP,
436                            mClock.getElapsedSinceBootMillis() + SCAN_TIMEOUT_MS,
437                            TIMEOUT_ALARM_TAG, mScanTimeoutListener, mEventHandler);
438                } else {
439                    Log.e(TAG, "Failed to start scan, freqs=" + freqs);
440                    // indicate scan failure async
441                    mEventHandler.post(new Runnable() {
442                            public void run() {
443                                if (newScanSettings.singleScanEventHandler != null) {
444                                    newScanSettings.singleScanEventHandler
445                                            .onScanStatus(WifiNative.WIFI_SCAN_FAILED);
446                                }
447                            }
448                        });
449                    // TODO(b/27769665) background scans should be failed too if scans fail enough
450                }
451            } else if (isHwPnoScanRequired()) {
452                newScanSettings.setHwPnoScan(mPnoSettings.networkList, mPnoEventHandler);
453                boolean status;
454                // If the PNO network list has changed from the previous request, ensure that
455                // we bypass the debounce logic and restart PNO scan.
456                if (isDifferentPnoScanSettings(newScanSettings)) {
457                    status = restartHwPnoScan(mPnoSettings);
458                } else {
459                    status = startHwPnoScan(mPnoSettings);
460                }
461                if (status) {
462                    mLastScanSettings = newScanSettings;
463                } else {
464                    Log.e(TAG, "Failed to start PNO scan");
465                    // indicate scan failure async
466                    mEventHandler.post(new Runnable() {
467                        public void run() {
468                            if (mPnoEventHandler != null) {
469                                mPnoEventHandler.onPnoScanFailed();
470                            }
471                            // Clean up PNO state, we don't want to continue PNO scanning.
472                            mPnoSettings = null;
473                            mPnoEventHandler = null;
474                        }
475                    });
476                }
477            }
478        }
479    }
```

11. scan
```java
// /frameworks/opt/net/wifi/service/java/com/android/server/wifi/WificondControl.java
369    /**
370     * Start a scan using wificond for the given parameters.
371     * @param freqs list of frequencies to scan for, if null scan all supported channels.
372     * @param hiddenNetworkSSIDs List of hidden networks to be scanned for.
373     * @return Returns true on success.
374     */
375    public boolean scan(Set<Integer> freqs, Set<String> hiddenNetworkSSIDs) {
376        if (mWificondScanner == null) {
377            Log.e(TAG, "No valid wificond scanner interface handler");
378            return false;
379        }
380        SingleScanSettings settings = new SingleScanSettings();
381        settings.channelSettings  = new ArrayList<>();
382        settings.hiddenNetworks  = new ArrayList<>();
383
384        if (freqs != null) {
385            for (Integer freq : freqs) {
386                ChannelSettings channel = new ChannelSettings();
387                channel.frequency = freq;
388                settings.channelSettings.add(channel);
389            }
390        }
391        if (hiddenNetworkSSIDs != null) {
392            for (String ssid : hiddenNetworkSSIDs) {
393                HiddenNetwork network = new HiddenNetwork();
394                try {
395                    network.ssid = NativeUtil.byteArrayFromArrayList(NativeUtil.decodeSsid(ssid));
396                } catch (IllegalArgumentException e) {
397                    Log.e(TAG, "Illegal argument " + ssid, e);
398                    continue;
399                }
400                settings.hiddenNetworks.add(network);
401            }
402        }
403
404        try {
405            return mWificondScanner.scan(settings); // http://androidxref.com/8.0.0_r4/xref/system/connectivity/wificond/scanning/scanner_impl.h#34
// http://androidxref.com/8.0.0_r4/xref/system/connectivity/wificond/scanning/scanner_impl.cpp#43
406        } catch (RemoteException e1) {
407            Log.e(TAG, "Failed to request scan due to remote exception");
408        }
409        return false;
410    }
411
```

12. 还是scan
```cpp
// http://androidxref.com/8.0.0_r4/xref/system/connectivity/wificond/scanning/scanner_impl.cpp#43

165Status ScannerImpl::scan(const SingleScanSettings& scan_settings,
166                         bool* out_success) {
167  if (!CheckIsValid()) {
168    *out_success = false;
169    return Status::ok();
170  }
171
172  if (scan_started_) {
173    LOG(WARNING) << "Scan already started";
174  }
175  // Only request MAC address randomization when station is not associated.
176  bool request_random_mac =  wiphy_features_.supports_random_mac_oneshot_scan &&
177      !client_interface_->IsAssociated();
178
179  // Initialize it with an empty ssid for a wild card scan.
180  vector<vector<uint8_t>> ssids = {{}};
181
182  vector<vector<uint8_t>> skipped_scan_ssids;
183  for (auto& network : scan_settings.hidden_networks_) {
184    if (ssids.size() + 1 > scan_capabilities_.max_num_scan_ssids) {
185      skipped_scan_ssids.emplace_back(network.ssid_);
186      continue;
187    }
188    ssids.push_back(network.ssid_);
189  }
190
191  LogSsidList(skipped_scan_ssids, "Skip scan ssid for single scan");
192
193  vector<uint32_t> freqs;
194  for (auto& channel : scan_settings.channel_settings_) {
195    freqs.push_back(channel.frequency_);
196  }
197
198  if (!scan_utils_->Scan(interface_index_, request_random_mac, ssids, freqs)) {
199    *out_success = false;
200    return Status::ok();
201  }
202  scan_started_ = true;
203  *out_success = true;
204  return Status::ok();
205}
206
```

13. 又是Scan
```cpp
// /system/connectivity/wificond/scanning/scan_utils.cpp

220bool ScanUtils::Scan(uint32_t interface_index,
221                     bool request_random_mac,
222                     const vector<vector<uint8_t>>& ssids,
223                     const vector<uint32_t>& freqs) {
224  NL80211Packet trigger_scan(
225      netlink_manager_->GetFamilyId(),
226      NL80211_CMD_TRIGGER_SCAN,
227      netlink_manager_->GetSequenceNumber(),
228      getpid());
229  // If we do not use NLM_F_ACK, we only receive a unicast repsonse
230  // when there is an error. If everything is good, scan results notification
231  // will only be sent through multicast.
232  // If NLM_F_ACK is set, there will always be an unicast repsonse, either an
233  // ERROR or an ACK message. The handler will always be called and removed by
234  // NetlinkManager.
235  trigger_scan.AddFlag(NLM_F_ACK);
236  NL80211Attr<uint32_t> if_index_attr(NL80211_ATTR_IFINDEX, interface_index);
237
238  NL80211NestedAttr ssids_attr(NL80211_ATTR_SCAN_SSIDS);
239  for (size_t i = 0; i < ssids.size(); i++) {
240    ssids_attr.AddAttribute(NL80211Attr<vector<uint8_t>>(i, ssids[i]));
241  }
242  NL80211NestedAttr freqs_attr(NL80211_ATTR_SCAN_FREQUENCIES);
243  for (size_t i = 0; i < freqs.size(); i++) {
244    freqs_attr.AddAttribute(NL80211Attr<uint32_t>(i, freqs[i]));
245  }
246
247  trigger_scan.AddAttribute(if_index_attr);
248  trigger_scan.AddAttribute(ssids_attr);
249  // An absence of NL80211_ATTR_SCAN_FREQUENCIES attribue informs kernel to
250  // scan all supported frequencies.
251  if (!freqs.empty()) {
252    trigger_scan.AddAttribute(freqs_attr);
253  }
254
255  if (request_random_mac) {
256    trigger_scan.AddAttribute(
257        NL80211Attr<uint32_t>(NL80211_ATTR_SCAN_FLAGS,
258                              NL80211_SCAN_FLAG_RANDOM_ADDR));
259  }
260  // We are receiving an ERROR/ACK message instead of the actual
261  // scan results here, so it is OK to expect a timely response because
262  // kernel is supposed to send the ERROR/ACK back before the scan starts.
263  vector<unique_ptr<const NL80211Packet>> response;
264  if (!netlink_manager_->SendMessageAndGetAck(trigger_scan)) {
265    LOG(ERROR) << "NL80211_CMD_TRIGGER_SCAN failed";
266    return false;
267  }
268  return true;
269}
```

14. SendMessageAndGetAck
```cpp
// /system/connectivity/wificond/net/netlink_manager.cpp
353bool NetlinkManager::SendMessageAndGetAck(const NL80211Packet& packet) {
354  int error_code;
355  if (!SendMessageAndGetAckOrError(packet, &error_code)) {

/*
	337bool NetlinkManager::SendMessageAndGetAckOrError(const NL80211Packet& packet,
	338                                                 int* error_code) {
	339  unique_ptr<const NL80211Packet> response;
	340  if (!SendMessageAndGetSingleResponseOrError(packet, &response)) {
	#if 0
		321bool NetlinkManager::SendMessageAndGetSingleResponseOrError(
		322    const NL80211Packet& packet,
		323    unique_ptr<const NL80211Packet>* response) {
		324  vector<unique_ptr<const NL80211Packet>> response_vec;
		325  if (!SendMessageAndGetResponses(packet, &response_vec)) { // 15
		326    return false;
		327  }
		328  if (response_vec.size() != 1) {
		329    LOG(ERROR) << "Unexpected response size: " << response_vec.size();
		330    return false;
		331  }
		332
		333  *response = std::move(response_vec[0]);
		334  return true;
		335}
	#endif 
	341    return false;
	342  }
	343  uint16_t type = response->GetMessageType();
	344  if (type != NLMSG_ERROR) {
	345    LOG(ERROR) << "Receive unexpected message type :" << type;
	346    return false;
	347  }
	348
	349  *error_code = response->GetErrorCode();
	350  return true;
	351}
*/

356    return false;
357  }
358  if (error_code != 0) {
359    LOG(ERROR) << "Received error messsage: " << strerror(error_code);
360    return false;
361  }
362
363  return true;
364}
```

15. SendMessageAndGetResponses
```cpp
254bool NetlinkManager::SendMessageAndGetResponses(
255    const NL80211Packet& packet,
256    vector<unique_ptr<const NL80211Packet>>* response) {
257  if (!SendMessageInternal(packet, sync_netlink_fd_.get())) {
258    return false;
259  }
260  // Polling netlink socket, waiting for GetFamily reply.
261  struct pollfd netlink_output;
262  memset(&netlink_output, 0, sizeof(netlink_output));
263  netlink_output.fd = sync_netlink_fd_.get();
264  netlink_output.events = POLLIN;
265
266  uint32_t sequence = packet.GetMessageSequence();
267
268  int time_remaining = kMaximumNetlinkMessageWaitMilliSeconds;
269  // Multipart messages may come with seperated datagrams, ending with a
270  // NLMSG_DONE message.
271  // ReceivePacketAndRunHandler() will remove the handler after receiving a
272  // NLMSG_DONE message.
273  message_handlers_[sequence] = std::bind(AppendPacket, response, _1);
274
275  while (time_remaining > 0 &&
276      message_handlers_.find(sequence) != message_handlers_.end()) {
277    nsecs_t interval = systemTime(SYSTEM_TIME_MONOTONIC);
278    int poll_return = poll(&netlink_output,
279                           1,
280                           time_remaining);
281
282    if (poll_return == 0) {
283      LOG(ERROR) << "Failed to poll netlink fd: time out ";
284      message_handlers_.erase(sequence);
285      return false;
286    } else if (poll_return == -1) {
287      LOG(ERROR) << "Failed to poll netlink fd: " << strerror(errno);
288      message_handlers_.erase(sequence);
289      return false;
290    }
291    ReceivePacketAndRunHandler(sync_netlink_fd_.get());
292    interval = systemTime(SYSTEM_TIME_MONOTONIC) - interval;
293    time_remaining -= static_cast<int>(ns2ms(interval));
294  }
295  if (time_remaining <= 0) {
296    LOG(ERROR) << "Timeout waiting for netlink reply messages";
297    message_handlers_.erase(sequence);
298    return false;
299  }
300  return true;
301}
302
```