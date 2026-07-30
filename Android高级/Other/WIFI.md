> 场景：
> 
> 1、扫描WI-FI设备
> 
> 2、连接到WI-FI设备
> 
> 3、无法从系统获取WI-FI密码，无法验证WI-FI密码是否正确

### 权限

普通权限：

```xml
<uses-permission android:name="android.permission.INTERNET"/>
    <uses-permission android:name="android.permission.ACCESS_WIFI_STATE"/>
    <uses-permission android:name="android.permission.CHANGE_WIFI_STATE"/>
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>
    <uses-permission android:name="android.permission.CHANGE_NETWORK_STATE"/>
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
```

运行时权限：

```kotlin
permissions =
                arrayOf(
                    Manifest.permission.BLUETOOTH_SCAN,
                    Manifest.permission.BLUETOOTH_CONNECT,
                    Manifest.permission.BLUETOOTH_ADVERTISE,
                    Manifest.permission.ACCESS_FINE_LOCATION,
                    Manifest.permission.CHANGE_NETWORK_STATE,
                    Manifest.permission.ACCESS_NETWORK_STATE
                )
```

### 扫描WI-FI

获取权限后，直接调用`mWifiManager.startScan()`进行扫描，三秒钟后通过`mWifiManager.scanResults`获取扫描结果。

```kotlin
object WifiManager {
    private val TAG = "WifiManager"
    private val mWifiManager: WifiManager by lazy {
        BaseApplication.instance.applicationContext.getSystemService(Context.WIFI_SERVICE) as WifiManager
    }

    /**
     * 扫描WI-FI信息
     */
    fun scanWifiInfo(callback: (ssid: String, encrypted: Boolean) -> Unit) {
        scan2_4GWifiResult.clear()
        mWifiManager.startScan()
        CoroutineScope(Dispatchers.IO).launch {
            delay(3000)
            val scanResults = mWifiManager.scanResults
            for ((index, scanResult) in scanResults.withIndex()) {
                //通过频率判断是否是2.4G网络
                val requency = scanResult.frequency
                val is24GHz = requency >= 2412 && requency <= 2484
                //scanResult.securityTypes直接返回WifiInfo的SECURITY_TYPE_*,但该版本不适配低版本
                //通过scanResult.capabilities判断WI-FI加密类型
                val wifiCapability = getCipherType(scanResult.capabilities)
                //是否有密码
                val encrypted = when(wifiCapability){
                    WifiCapability.WIFI_CIPHER_NONE->false
                    else->true
                }
                "wifi info ssid:${scanResult.SSID} is24GHz:$is24GHz encrypted:$encrypted encryptType:${wifiCapability}".logd(TAG)
                if (is24GHz) {
                    scan2_4GWifiResult.add(scanResult)
                    callback(scanResult.SSID, encrypted)
                }
            }
        }
    }

    //解析密码加密类型
    private fun getCipherType(capabilities: String): WifiCapability = when {
        capabilities.contains("SAE") -> WifiCapability.WIFI_CIPHER_WPA3
        capabilities.contains("WEP") -> WifiCapability.WIFI_CIPHER_WEP
        capabilities.contains("PSK") -> WifiCapability.WIFI_CIPHER_WPA2
        capabilities.contains("EAP") -> WifiCapability.WIFI_CIPHER_EAP
        capabilities.contains("OWE") -> WifiCapability.WIFI_CIPHER_OWE
        else -> WifiCapability.WIFI_CIPHER_NONE
    }

    enum class WifiCapability {
        //没有密码：SECURITY_TYPE_OPEN = 0
        WIFI_CIPHER_NONE,

        //WEP加密：SECURITY_TYPE_WEP = 1
        WIFI_CIPHER_WEP,

        //WPA-PSK/WPA2-PSK加密：SECURITY_TYPE_PSK = 2;
        WIFI_CIPHER_WPA2,

        //企业级EAP认证：SECURITY_TYPE_EAP = 3
        WIFI_CIPHER_EAP,

        //WPA3-SAE (从Android 11开始)：SECURITY_TYPE_SAE = 4;
        WIFI_CIPHER_WPA3,

        //OWE (Opportunistic Wireless Encryption):SECURITY_TYPE_OWE = 5
        WIFI_CIPHER_OWE,

        //政府/企业级加密：SECURITY_TYPE_SUITE_B = 6
        WIFI_CIPHER_SUITE_B
    }

}
```

### 连接到WI-FI

连接WI-FI要区分不同的版本，Android Q及以上版本使用P2P连接，手机会弹出建议连接的弹窗。Android Q以下通过向系统添加`WifiConfiguration`进行连接。

对于不同的加密类型，设置也不一样。

AndroidQ及以上P2P连接成功后，还需要将网络绑定到进程，网络才可以使用

```kotlin
//Android10以上，通过P2P连接WI-FI
    @RequiresApi(Build.VERSION_CODES.Q)
    private fun connectByP2P(ssid: String, pwd: String,capability: WifiCapability) {
        "connectByP2P".logd(TAG)
        val specifier = WifiNetworkSpecifier.Builder()
            .setSsid(ssid)  //设置ssid
            //根据不同的加密模式设置pwd，只判断WPA2和WPA3两种模式
            .apply {
                when(capability){
                    WifiCapability.WIFI_CIPHER_WPA2->this.setWpa2Passphrase(pwd)
                    WifiCapability.WIFI_CIPHER_WPA3->this.setWpa3Passphrase(pwd)
                    else->{}
                }
            }
            .build()
        val request = NetworkRequest.Builder()
            .addTransportType(NetworkCapabilities.TRANSPORT_WIFI)
            .setNetworkSpecifier(specifier)
//            .addCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
//            .addCapability(NetworkCapabilities.NET_CAPABILITY_VALIDATED)  //NET_CAPABILITY_VALIDATED无法直接添加
            .build()
        //网络回调
        val networkCallback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) {
                super.onAvailable(network)
                "connect $ssid success".logd(TAG)
                //绑定网络到进程
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
                    mConnectivityManager.bindProcessToNetwork(network)
                } else {
                    ConnectivityManager.setProcessDefaultNetwork(network)
                }
            }

            override fun onUnavailable() {
                super.onUnavailable()
                "connect $ssid failure".loge(TAG)
            }
        }
        mConnectivityManager.requestNetwork(request, networkCallback)
    }
```

Android Q以下：

```kotlin
private val mWifiManager: WifiManager by lazy {
        BaseApplication.instance.applicationContext.getSystemService(Context.WIFI_SERVICE) as WifiManager
    }
private val mConnectivityManager: ConnectivityManager by lazy {
        BaseApplication.instance.applicationContext.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
    }

fun connectWifiLegacy(ssid: String, pwd: String,capability: WifiCapability){
     //扫描结果
        val scanResult = scan2_4GWifiResult.singleOrNull { it.SSID == ssid }
        if (scanResult == null) {
            "not wifi ssid is $ssid".toastWith(BaseApplication.instance.baseContext)
            return
        }

        var success = false
        //复用已有的Config
        val config =
            mWifiManager.configuredNetworks.singleOrNull { it.SSID.replace("\"", "") == ssid }
        success = if (config != null) {
            mWifiManager.enableNetwork(config.networkId, true)
        } else {
            //创建新的Config
            val padWifiNetwork = createWifiConfig(ssid, pwd, getCipherType(scanResult.capabilities))
            val netId = mWifiManager.addNetwork(padWifiNetwork)
            mWifiManager.enableNetwork(netId, true)
        }
        "connect wifi $ssid is ${if (success) "success" else "failuer"}".toastWith(BaseApplication.instance.baseContext)
}


//创建一个WI-FI配置
    fun createWifiConfig(ssid: String, pwd: String, type: WifiCapability): WifiConfiguration {
        //初始化WifiConfiguration
        val config = WifiConfiguration().apply {
            allowedAuthAlgorithms.clear()
            allowedGroupCiphers.clear()
            allowedKeyManagement.clear()
            allowedPairwiseCiphers.clear()
            allowedProtocols.clear()
            SSID = "\"$ssid\""   //指定SSID

        }
        //查找是否有类似的配置
        val tempConfig = mWifiManager.configuredNetworks.singleOrNull { it.SSID == "\"$ssid\"" }
        //清除旧有配置
        if (tempConfig != null) {
            mWifiManager.removeNetwork(tempConfig.networkId)
            mWifiManager.saveConfiguration()
        }

        when (type) {
            //不需要密码的场景
            WifiCapability.WIFI_CIPHER_NONE -> config.allowedKeyManagement.set(WifiConfiguration.KeyMgmt.NONE)
            //WEP加密场景
            WifiCapability.WIFI_CIPHER_WEP -> {
                config.run {
                    hiddenSSID = true
                    wepKeys[0] = "\"$pwd\""
                    allowedAuthAlgorithms.set(WifiConfiguration.AuthAlgorithm.OPEN)
                    allowedAuthAlgorithms.set(WifiConfiguration.AuthAlgorithm.SHARED)
                    allowedKeyManagement.set(WifiConfiguration.KeyMgmt.NONE)
                    wepTxKeyIndex = 0
                }
            }
            //WPA加密场景
            WifiCapability.WIFI_CIPHER_WPA2 -> {
                config.run {
                    preSharedKey = "\"$pwd\""
                    hiddenSSID = true
                    allowedAuthAlgorithms.set(WifiConfiguration.AuthAlgorithm.OPEN)
                    allowedKeyManagement.set(WifiConfiguration.KeyMgmt.WPA_PSK)
                    allowedGroupCiphers.set(WifiConfiguration.GroupCipher.TKIP)
                    allowedGroupCiphers.set(WifiConfiguration.GroupCipher.CCMP)
                    allowedPairwiseCiphers.set(WifiConfiguration.PairwiseCipher.TKIP)
                    allowedPairwiseCiphers.set(WifiConfiguration.PairwiseCipher.CCMP)
                    status = WifiConfiguration.Status.ENABLED
                }
            }

            else -> {}
        }
        return config
    }
```
