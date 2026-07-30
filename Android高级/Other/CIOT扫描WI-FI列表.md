## CIOT 扫描WI-FI列表

*无法扫描WI-FI列表问题*

1. 在升级Android SDK后，VIVO手机（Android15）调用`WifiManager.startScan()`返回False，无法扫描到WI-FI设备。同样是Android15的小米手机，能够扫描到WI-FI设备。需要进行厂商适配。

2. 鸿蒙2.0（荣耀20），鸿蒙4.2（华为Mate60），已开启附近设备权限和定位权限，但应用内检测到的仍是没有获取对应权限，无法扫描WI-FI列表。



### 明确扫描WI-FI功能各个SDK版本权限

1. Android8.0、Android8.1，需要以下任一权限
   
   - [`ACCESS_FINE_LOCATION`](https://developer.android.google.cn/reference/android/Manifest.permission?hl=zh-cn#ACCESS_FINE_LOCATION)
   
   - [`ACCESS_COARSE_LOCATION`](https://developer.android.google.cn/reference/android/Manifest.permission?hl=zh-cn#ACCESS_COARSE_LOCATION)
   
   - [`CHANGE_WIFI_STATE`](https://developer.android.google.cn/reference/android/Manifest.permission?hl=zh-cn#CHANGE_WIFI_STATE)

2. Android9
   
   -  [`ACCESS_FINE_LOCATION`](https://developer.android.google.cn/reference/android/Manifest.permission?hl=zh-cn#ACCESS_FINE_LOCATION)或 [`ACCESS_COARSE_LOCATION`](https://developer.android.google.cn/reference/android/Manifest.permission?hl=zh-cn#ACCESS_COARSE_LOCATION)
   
   -  [`CHANGE_WIFI_STATE`](https://developer.android.google.cn/reference/android/Manifest.permission?hl=zh-cn#CHANGE_WIFI_STATE)
   
   - 设备已启用位置信息服务（位于**设置 > 位置信息**下）

3. Android10(API 29)以上
   
   - [`ACCESS_FINE_LOCATION`](https://developer.android.google.cn/reference/android/Manifest.permission?hl=zh-cn#ACCESS_FINE_LOCATION)
   
   - [`CHANGE_WIFI_STATE`](https://developer.android.google.cn/reference/android/Manifest.permission?hl=zh-cn#CHANGE_WIFI_STATE)
   
   - 设备已启用位置信息服务（位于**设置** > **位置信息**下）



【网络不好的时候VIVO无法获取WI-FI列表】


