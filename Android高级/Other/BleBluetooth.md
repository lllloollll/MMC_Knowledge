蓝牙与低功耗蓝牙

BLE低功耗蓝牙，显著降低功耗。

> **注意**：当用户使用 BLE 将其设备与其他设备配对时，用户设备上的**所有**应用都可以访问在这两个设备间传输的数据。
> 
> 因此，如果您的应用捕获敏感数据，您应实现应用层安全以保护此类数据的私密性。

#### 建立通信通道前提

1、声明权限

2、





#### 实现

> 实现扫描周围蓝牙设备，并且连接到蓝牙

*在实现IOT设备联网功能时，使用手机蓝牙功能主动扫描周围设备，并且通过蓝牙连接到设备，和设备进行数据通讯*

实现手机扫描周围蓝牙设备，并且连接到蓝牙设备，需要相应的权限

#### 需要的权限

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- 蓝牙的权限 -->
    <!-- Android 11 及以下版本 -->
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission
        android:name="android.permission.BLUETOOTH"
        android:maxSdkVersion="30" />
    <uses-permission
        android:name="android.permission.BLUETOOTH_ADMIN"
        android:maxSdkVersion="30" />
    <!-- Android 12 及以上版本 ，运行时权限 -->
    <uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
    <uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
    <uses-permission android:name="android.permission.BLUETOOTH_ADVERTISE" />
</manifest>
```

运行时权限

```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
            permissions =
                arrayOf(
                    Manifest.permission.BLUETOOTH_SCAN,
                    Manifest.permission.BLUETOOTH_CONNECT,
                    Manifest.permission.BLUETOOTH_ADVERTISE,
                    Manifest.permission.ACCESS_FINE_LOCATION
                )
        } else {
            permissions = arrayOf(Manifest.permission.ACCESS_FINE_LOCATION)
        }
```

获取权限后，进行蓝牙扫描工作

#### 蓝牙扫描功能

```kotlin
//从上下文创建蓝牙管理类
val bluetoothManager: BluetoothManager by lazy { context.getSystemService(BluetoothManager::class.java) }
//直接获取蓝牙适配器
val bluetoothAdapter: BluetoothAdapter? by lazy { bluetoothManager.adapter }
//从蓝牙适配器获取蓝牙扫描器
val bluetoothLeScanner: BluetoothLeScanner? by lazy { bluetoothAdapter?.bluetoothLeScanner }
//扫描回调
val leScanCallback: ScanCallback = object : ScanCallback() {
        override fun onScanResult(callbackType: Int, result: ScanResult?) {
            super.onScanResult(callbackType, result)
            result?.let {
                //扫描到的设备
                val device = it.device
                //设备名称
                val deviceName = device?.name?.toString()
                //信息
                val scanRecord = result.scanRecord?.bytes
            }
        }
    }
//开始扫描
try {
                //过滤器，设置过滤设备名称name
            val scanFilter1 = ScanFilter.Builder().setDeviceName(name).build()
            val scanSetting = ScanSettings.Builder().build()
            GlobalScope.launch {
                //开始扫描，12000ms后停止扫描
                bluetoothLeScanner?.startScan(
                    Arrays.asList(scanFilter1),
                    scanSetting,
                    leScanCallback
                )
                delay(12000)
                bluetoothLeScanner?.stopScan(leScanCallback)
            }
        } catch (e: IllegalArgumentException) {
            e.printStackTrace()
        }
```
