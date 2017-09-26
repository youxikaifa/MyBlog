---
title: bluetoothGaTT详解
date: 2017-09-26 12:44:24
tags:
---
本文将展开对蓝牙低功耗从扫描蓝牙设备，建立连接到蓝牙数据通信的详细介绍，以及详细介绍[GATT Profile](https://www.race604.com/gatt-profile-intro/)(Generic Attribute Profile，通用属性协议)的组成结构。

###### 权限和feature
  和经典蓝牙一样，使用低功耗蓝牙，需要声明BLUETOOTH权限，如果需要扫描设备或者操作蓝牙设置，则还需要BLUETOOTH_ADMIN权限：
```
<uses-p1ermission android:name="android.permission.BLUETOOTH"/>
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN"/>
```

除了蓝牙权限外，如果需要BLE feature则还需要声明uses-feature：

```
<uses-feature android:name="android.hardware.bluetooth_le" android:required="true"/>
```

当required为true时，则应用只能在支持BLE的Android设备上安装运行；required为false时，Android设备均可正常安装运行，需要在代码中判断设备是否支持BLE feature：

```
if(!getPackageManager().hasSystemFeature(PackageManager.FEATURE_BLUETOOTH_LE)) {
  Toast.makeText(this, R.string.ble_not_supported, Toast.LENGTH_SHORT).show();
  finish();
}
```
###### 创建BLE
在应用可以通过 BLE 交互之前, 你需要验证设备是否支持 BLE 功能, 如果支持, 确定它是可以使用的. 这个检查只有在 下面的配置 设置为 false 时才是必须的。

```
<uses-feature android:name="android.hardware.bluetooth_le" android:required="true"/>
```
###### 获取蓝牙适配器（BluetoothAdapter）
所有的蓝牙活动都需要 BluetoothAdapter, BluetoothAdapter 代表了设备本身的蓝牙适配器 (蓝牙无线设备). 整个系统中只有一个 蓝牙适配器, 应用可以使用 BluetoothAdapter 对象与 蓝牙适配器硬件进行交互.
```
// Initializes a Bluetooth adapter.  For API level 18 and above, get a reference to
// BluetoothAdapter through BluetoothManager.
final BluetoothManager bluetoothManager = (BluetoothManager) getSystemService(Context.BLUETOOTH_SERVICE);
mBluetoothAdapter = bluetoothManager.getAdapter();
```
###### 打开蓝牙功能
  为了保证 蓝牙功能是打开的, 调用 BluetoothAdapter 的 isEnable() 方法, 检查蓝牙在当前是否可用. 如果返回 false, 说明当前蓝牙不可用.

```
// 确认当前设备的蓝牙是否可用,   
// 如果不可用, 弹出一个对话框, 请求打开设备的蓝牙模块  
if (mBluetoothAdapter == null || !mBluetoothAdapter.isEnabled()) {  
    Intent enableBtIntent = new Intent(BluetoothAdapter.ACTION_REQUEST_ENABLE);  
    startActivityForResult(enableBtIntent, REQUEST_ENABLE_BT);  
}  
```

###### 查找设备
搜索 BLE 设备, 调用 BluetoothAdapter 的 startLeScan() 方法, 该方法需要一个 BluetoothAdapter.LeScanCallback 类型的参数. 你必须实现这个 LeScanCallback 接口, 因为 BLE 蓝牙设备扫描结果在这个接口中返回.蓝牙搜索是一个耗电的操作，因此蓝牙搜索提供了两种搜索模式：
*中断搜索*：一搜索到设备就中断搜索
*不循环搜索*：给搜索设置一个合适的扫描周期

扫描设备：

```
private void scanLeDevice(final boolean enable) {
        if (enable) {
            // Stops scanning after a pre-defined scan period.
            mHandler.postDelayed(new Runnable() {

                public void run() {
                    isScanned = false;
                    refreshLayout.setRefreshing(false);
                    mBluetoothAdapter.stopLeScan(mLeScanCallback);
                }
            }, SCAN_PERIOD);

            isScanned = true;
            mBluetoothAdapter.startLeScan(mLeScanCallback);
        } else {
            if (refreshLayout.isRefreshing()){
                refreshLayout.setRefreshing(false);
            }
            isScanned = false;
            mBluetoothAdapter.stopLeScan(mLeScanCallback);
        }
    }
```
回调代码：
```
 private final LeScanCallback mLeScanCallback = new LeScanCallback() {

        public void onLeScan(final BluetoothDevice device, int rssi, byte[] scanRecord) {
            runOnUiThread(new Runnable() {
                public void run() {
                    if (!devices.contains(device)){
                        devices.add(device);
                        adapter.notifyDataSetChanged();
                    }

                }
            });
        }
    };
```
###### 查找特定设备
查找特定类型的外围设备, 可以调用下面的方法, 这个方法需要提供一个 UUID 对象数组, 这个 UUID 数组是 APP 支持的 GATT 服务的特殊标识.

```
startLeScan(UUID[], BluetoothAdapter.LeScanCallback)  
```

###### 连接到GATT服务
调用 BluetoothDevice 的 connectGatt() 方法可以连接到 BLE 设备的 GATT 服务. 参数一 Context 上下文对象, 参数二 boolean autoConnect 是否自动连接扫描到的蓝牙设备, 参数三 BluetoothGattCallback 接口实现类.

```
mBluetoothGatt = device.connectGatt(this, false, mGattCallback);  
```
###### GATT数据交互
这段代码的本质就是 BLE 设备的 GATT 服务 与 Android 的 BLE API 进行交流. 当一个特定的回调被触发, 它调用适当的 broadcastUpdate() 帮助方法, 将其当做一个 Action 操作传递出去.   

```
/**
     * Implements callback methods for GATT events that the app cares about.
     * For example,connection change and services discovered.
     */
    private final BluetoothGattCallback mGattCallback = new BluetoothGattCallback() {
        @Override
        public void onConnectionStateChange(BluetoothGatt gatt, int status, int newState) {
            String intentAction;
            if (newState == BluetoothProfile.STATE_CONNECTED) {
                intentAction = ACTION_GATT_CONNECTED;
                broadcastUpdate(intentAction);
                Log.i(TAG, "Connected to GATT server.");
                // Attempts to discover services after successful connection.
                Log.i(TAG, "Attempting to start service discovery:" + mBluetoothGatt.discoverServices());

            } else if (newState == BluetoothProfile.STATE_DISCONNECTED) {
                intentAction = ACTION_GATT_DISCONNECTED;
                Log.i(TAG, "Disconnected from GATT server.");
                broadcastUpdate(intentAction);
            }else if (newState == BluetoothProfile.STATE_CONNECTING){
                intentAction = ACTION_GATT_CONNECTING;
                Log.i(TAG, "connecting from GATT server.");
                broadcastUpdate(intentAction);
            }else{
                intentAction = ACTION_GATT_DISCONNECTING;
                Log.i(TAG, "Disconnecting from GATT server.");
                broadcastUpdate(intentAction);
            }
        }
```
###### 读取 BLE 属性
  到了这里我就要详细的讲解一下BLE GATT PROFILE的组织结构了。每个蓝牙设备都有一个Profile（就把这个Profile想象成是一个蓝牙模块），每个Profile有多个service（服务）如电量信息服务、系统信息服务等，每个service有多个Characteristic（特征），每个特征里面包括属性（properties）和值（value）和若干个descriptor（描述符）。我们刚才提到的service和characteristic，都需要一个唯一的uuid来标识，如图：
![GATT Profile 结构图](https://github.com/youxikaifa/jianshuRes/blob/master/Android%20BLE%20%E8%BF%9E%E6%8E%A5%E5%8F%8A%E6%95%B0%E6%8D%AE%E4%BC%A0%E8%BE%93%E8%AF%A6%E8%A7%A3/gatt.png?raw=true)
Android 应用连接到了 设备中的 GATT 服务, 并且发现了 各种服务 (特征集合), 可以读写其中的属性,遍历服务 (特征集合) 和 特征, 将其展示在 UI 界面中.

```
 // 遍历 GATT 服务  
        for (BluetoothGattService gattService : gattServices) {  
            HashMap<String, String> currentServiceData =  
                    new HashMap<String, String>();  
            uuid = gattService.getUuid().toString();  
            currentServiceData.put(  
                    LIST_NAME, SampleGattAttributes.  
                            lookup(uuid, unknownServiceString));  
            currentServiceData.put(LIST_UUID, uuid);  
            gattServiceData.add(currentServiceData);  

            ArrayList<HashMap<String, String>> gattCharacteristicGroupData =  
                    new ArrayList<HashMap<String, String>>();  

            // 获取服务中的特征集合  
            List<BluetoothGattCharacteristic> gattCharacteristics =  
                    gattService.getCharacteristics();  
            ArrayList<BluetoothGattCharacteristic> charas =  
                    new ArrayList<BluetoothGattCharacteristic>();  

           // 循环遍历特征集合  
            for (BluetoothGattCharacteristic gattCharacteristic :  
                    gattCharacteristics) {  
                charas.add(gattCharacteristic);  
                HashMap<String, String> currentCharaData =  
                        new HashMap<String, String>();  
                uuid = gattCharacteristic.getUuid().toString();  
                currentCharaData.put(  
                        LIST_NAME, SampleGattAttributes.lookup(uuid,  
                                unknownCharaString));  
                currentCharaData.put(LIST_UUID, uuid);  
                gattCharacteristicGroupData.add(currentCharaData);  
            }  
            mGattCharacteristics.add(charas);  
            gattCharacteristicData.add(gattCharacteristicGroupData);  
         }  
```
###### 接收GATT通知
当 BLE 设备中的一些特殊的特征改变, 需要通知与之连接的 Android BLE 应用，使用 setCharacteristicNotification() 方法为特征设置通知.

```
private BluetoothGatt mBluetoothGatt;  
BluetoothGattCharacteristic characteristic;  
boolean enabled;  
...  
// 设置是否监听某个特征改变  
mBluetoothGatt.setCharacteristicNotification(characteristic, enabled);  
...  
BluetoothGattDescriptor descriptor = characteristic.getDescriptor(  
        UUID.fromString(SampleGattAttributes.CLIENT_CHARACTERISTIC_CONFIG));  
descriptor.setValue(BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE);  
mBluetoothGatt.writeDescriptor(descriptor);  
```
一旦特征开启了改变通知监听, 如果特性发生了改变, 就会回调 BluetoothGattCallback 接口中的 onCharacteristicChanged() 方法.

```
@Override  
// 特性通知  
public void onCharacteristicChanged(BluetoothGatt gatt,  
        BluetoothGattCharacteristic characteristic) {  
    broadcastUpdate(ACTION_DATA_AVAILABLE, characteristic);  
}  
```

###### 关闭APP中的BLE连接
一旦结束了 BLE 设备的使用, 调用 BluetoothGatt 的 close() 方法, 关闭 BLE 连接, 释放相关的资源.
```
public void close() {  
    if (mBluetoothGatt == null) {  
        return;  
    }  
    mBluetoothGatt.close();  
    mBluetoothGatt = null;  
}  
```

完整项目链接，访问我的开源中国账号[OSGIT]()
