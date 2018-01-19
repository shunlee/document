# DCS 3.0 轻量级 Android SDK V0.0.1 文档

## 1 SDK概述

### 1.1 SDK功能概述
SDK包含语音交互、数据点控制、自定义录音设备语音传输等接口。SDK有一个demo app，开发者可以参DemoApp开发自己的App。

## 2 集成准备 

### 2.1  Android Studio集成

复制我们提供的sdk（lightduerlib-release.aar）到项目/app/libs/目录 
在gradle文件中添加依赖

```
repositories{
    flatDir {
        dirs 'libs'
    }
}

dependencies {
    compile(name: 'lightduerlib-release', ext: 'aar')
}
```

### 2.2 profile申请

使用SDK需要profile文件，此文件中包含了：APPID，CUID等信息以及各种证书。可以在百度开放平台申请，网址：https://dueros.baidu.com/open

### 2.3  AndroidManifest.xml配置

在AndroidManifest.xml添加如下配置
```
    <uses-permission android:name="android.permission.RECORD_AUDIO" />
    <uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <uses-permission android:name="android.permission.CHANGE_WIFI_STATE" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.MOUNT_UNMOUNT_FILESYSTEMS" />
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.READ_PHONE_STATE" />
```

### 2.4  proguard规则
添加如下：
```
-keep class com.baidu.lightduer.** { *; }
```

## 3 SDK功能 

### 3.1  整体结构介绍
- DuerLightOSSDK：SDK总管理类，通过此类获取sdk对象。
- ClientImpl: 客户端API管理类,此对象通过DuerLightOSSDK类获取，通过此类初始化SDK、获取各类API访问实例。
- LightClient.IConnectStatusListener： 连接duer服务监听器（开启/停止）。
- LightClient.VoiceOutputListener：语音上传后，TTS结果返回监听器，返回tts的url。
- LightClient.AudioPlayerListener: 语音上传后，音频结果返回监听器，返回音频的url。
- LightClient.IAudioInputListener：语音输入状态监听，分别有正在录音、录音结束、录音错误。
- CustomAudioRecord： 用户自定义的录音器，需要代码中自行调用上传语音，具体用法见接口说明。 
- LightduerResource： 数据点类，用户在开放平台创建的数据点可以通过此类定义。

### 3.2  接口说明

#### 3.2.1  SDK初始化

初始化SDK分两步完成：1.initSdk  2.connectServer

#### 3.2.1.1  init sdk

**接口功能描述**

初始化SDK,在自定义MainActivity中调用如下方法,保证在调用LightDuerSDK其他接口前完成初始化,此方法可以在主线中调用。

**前置条件** 

无

**传入参数**

无

**回调函数** 

无

**实例**

```
DuerLightOSSDK.getInstance().getClient().init();
```

#### 3.2.1.2 connectServer

**接口功能描述**

连接lightduer server，在初始化SDK后连接duer server,此方法可以在主线中调用。

**前置条件** 

初始化SDK（DuerLightOSSDK.getInstance().getClient().init()）后调用;

**传入参数**

|类型|名称|描述|
|-|-|-|
|String|profile|profile文件，需要在开放平台上申请获取，读取profile文件转为String|
|LightClient.IConnectStatusListener|iConnectStatusListener|请求回调|

**回调函数** 

回调参数LightClient.IConnectStatusListener将在回调方法onConnectStatus(LightduerEvent lightduerEvent)中传递,lightduerEvent.getEventid()为LightduerEventId.DUER_EVENT_STARTED表示服务已经启动，其他状态则为启动失败。

**实例**

```
DuerLightOSSDK.getInstance().getClient().connectServer(profileSB.toString(), iConnectStatusListener);
LightClient.IConnectStatusListener iConnectStatusListener = new LightClient.IConnectStatusListener() {
    @Override
    public void onConnectStatus(LightduerEvent lightduerEvent) {
        if (lightduerEvent.getEventid() == LightduerEventId.DUER_EVENT_STARTED) {
            Toast.makeText(LightDuerMainActivity.this, R.string.connect_success, Toast.LENGTH_LONG).show();
        } else {
            Toast.makeText(LightDuerMainActivity.this, R.string.connect_fail, Toast.LENGTH_LONG).show();
        }
    }
};
```

#### 3.2.2 设置并初始化监听器
#### 3.2.2.1 setAudioInputListener

**接口功能描述**

设置此监听器用于监听当前录音器的状态：开始录音、录音停止、录音错误

**前置条件** 

无

**传入参数**

|类型|名称|描述|
|-|-|-|
|LightClient.IAudioInputListener|IAudioInputListener|输入语音状态回调|

**回调函数** 

有三个回调函数：
1. onStartRecord 设备开始录音
2. onStopRecord 设备停止录音
3. onErrorRecord(int errorCode) 设备录音发生错误。errorCode目前只有ClientImpl.ERROR_AUDIO_NO_PERMISSION，此code表示打开录音器没有权限。

**实例**

```
DuerLightOSSDK.getInstance().getClient().setAudioInputListener(new LightClient.IAudioInputListener() {
    @Override
    public void onStartRecord() {
    }

    @Override
    public void onStopRecord() {
    }

    @Override
    public void onErrorRecord(int errorCode) {
        Log.d(TAG, "record error code = " + errorCode);
        if (ClientImpl.ERROR_AUDIO_NO_PERMISSION == errorCode) {
            ActivityCompat.requestPermissions(LightDuerMainActivity.this, new String[]{Manifest.permission.RECORD_AUDIO}, 1);
        }
    }
});
```

#### 3.2.2.2 setVoiceOutputListener

**接口功能描述**

此监听器用于获取语音上传后的tts返回，返回的数据为tts的url。

**前置条件** 

无

**传入参数**

|类型|名称|描述|
|-|-|-|
|LightClient.VoiceOutputListener|VoiceOutputListener|返回的tts结果回调|

**回调函数** 

回调函数为：speak(String url)，url为上传语音后返回tts的url地址。

**实例**

```
DuerLightOSSDK.getInstance().getClient().setVoiceOutputListener(new LightClient.VoiceOutputListener() {
    @Override
    public int speak(String url) {
        return 0;
    }
});
```

#### 3.2.2.3 setAudioPlayerListener

**接口功能描述**

此监听器用于获取语音上传后的音频数据的返回，返回的数据为音频的url。

**前置条件** 

无

**传入参数**

|类型|名称|描述|
|-|-|-|
|LightClient.AudioPlayerListener|AudioPlayerListener|返回音频结果回调|

**回调函数** 

目前支持的回调函数有：int play(String url) {}，此url为音频数据的url地址。其他回调函数暂时不支持。

**实例**

```
DuerLightOSSDK.getInstance().getClient().setAudioPlayerListener(new LightClient.AudioPlayerListener() {
    @Override
    public int play(String url) {
        resultTv.setText("(AudioPlayerListener)url = " + url);
        mMediaPlayer.playUrl(url, LightDuerMediaPlayer.TYPE_AUDIO);
        return 0;
    }

    @Override
    public int stop() {
        return 0;
    }

    @Override
    public int resume(String url, int offset) {
        return 0;
    }

    @Override
    public int pause() {
        return 0;
    }

    @Override
    public int getAudioPlayProgress() {
        return 0;
    }
});
```

#### 3.2.3 数据点相关接口

添加数据点，需要在开放平台上添加对应的数据点，可以用于控制设备。

#### addControlPoint

**接口功能描述**

添加数据点，在代码中创建出对应的数据点添加对数据点的监听。

**前置条件** 

无

**传入参数**

|类型|名称|描述|
|-|-|-|
|LightduerResource |resources|数据点|

LightduerResource详细说明：

```
public LightduerResource(int mode, int allowed, String path, byte[] staticData, LightduerResourceListener callback) {}
```

int mode：   数据点的类型动态(LightduerResource.DUER_RES_MODE_DYNAMIC)和静态(LightduerResource.DUER_RES_MODE_STATIC)两种类型。  
int allowed：有四种类型{DUER_RES_OP_GET, DUER_RES_OP_PUT, DUER_RES_OP_POST, DUER_RES_OP_DELETE}  
String path：   开放平台上设置的数据点名称  
byte[] staticData：动态数据点可以传null，静态数据点传入静态数据点string获取到的bytes即可（static_resource.getBytes()）  
LightduerResourceListener callback：数据点的监听器，内有一个监听函数

```
public int callback(LightduerContext context, LightduerMessage message, LightduerAddress address) {}
```

LightduerContext context： 设置数据点的上下文  
LightduerMessage message： 数据点中返回的内容，可以通过message.getPayload()获取  
LightduerAddress address： 区分message是从那个地址过来的

**回调函数** 

无

**实例**

```
DuerLightOSSDK.getInstance().getClient().addControlPoint(resources);
```

#### 3.2.4 Android通用Record（非CustomAudioRecord）相关接口
使用sdk封装好的一套录音上传语音数据完成语音交互。

#### 3.2.4.1 startRecord

**接口功能描述**

开始语音交互。

**前置条件** 

无

**传入参数**

无

**回调函数** 

无

**实例**

```
DuerLightOSSDK.getInstance().getClient().startRecord();
```

#### 3.2.5 CustomAudioRecord相关接口

此接口为用户自定义audiorecord来完成语音录制并上传，需要注意的是此接口完全使用另一套流程。不需要再调用DuerLightOSSDK.getInstance().getClient().startRecord();接口开始语音交互。

#### 3.2.5.1 getCustomAudioRecord

**接口功能描述**

获取一个自定义的播放器。

**前置条件** 

无

**传入参数**

无

**回调函数** 

无

**实例**

```
DuerLightOSSDK.getInstance().getClient().getCustomAudioRecord();
```

#### 3.2.5.2 getAudioStatus

**接口功能描述**

获取当前的audioRecorder的状态：停止状态、开始状态。

**前置条件** 

无

**传入参数**

无

**回调函数** 

无

**返回值** 

|类型|名称|描述|
|-|-|-|
|CustomAudioRecord.Status |status|CustomAudioRecord.Status.STARTED 开始状态；CustomAudioRecord.Status.STOPED 停止状态|

**实例**

```
mCustomAudioRecord.getAudioStatus();
```

#### 3.2.5.3 startRecord

**接口功能描述**

在调用传输接口先调用此接口来通知sdk接受语音数据。

**前置条件** 

无

**传入参数**

|类型|名称|描述|
|-|-|-|
|int|samplerate|采样率，目前支持16000|

**回调函数** 

无

**实例**

```
mCustomAudioRecord.startRecord(16000);
```
#### 3.2.5.4 sendVoiceData

**接口功能描述**

向sdk发送语音数据。

**前置条件** 

先调用mCustomAudioRecord.# DCS 3.0 轻量级 Android SDK V0.0.1 文档

## 1 SDK概述

### 1.1 SDK功能概述
SDK包含语音交互、数据点控制、自定义录音设备语音传输等接口。SDK有一个demo app，开发者可以参DemoApp开发自己的App。

## 2 集成准备 

### 2.1  Android Studio集成

复制我们提供的sdk（lightduerlib-release.aar）到项目/app/libs/目录 
在gradle文件中添加依赖

```
repositories{
    flatDir {
        dirs 'libs'
    }
}

dependencies {
    compile(name: 'lightduerlib-release', ext: 'aar')
}
```

### 2.2 profile申请

使用SDK需要profile文件，此文件中包含了：APPID，CUID等信息以及各种证书。可以在百度开放平台申请，网址：https://dueros.baidu.com/open

### 2.3  AndroidManifest.xml配置

在AndroidManifest.xml添加如下配置
```
    <uses-permission android:name="android.permission.RECORD_AUDIO" />
    <uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <uses-permission android:name="android.permission.CHANGE_WIFI_STATE" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.MOUNT_UNMOUNT_FILESYSTEMS" />
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.READ_PHONE_STATE" />
```

### 2.4  proguard规则
添加如下：
```
-keep class com.baidu.lightduer.** { *; }
```

## 3 SDK功能 

### 3.1  整体结构介绍
- DuerLightOSSDK：SDK总管理类，通过此类获取sdk对象。
- ClientImpl: 客户端API管理类,此对象通过DuerLightOSSDK类获取，通过此类初始化SDK、获取各类API访问实例。
- LightClient.IConnectStatusListener： 连接duer服务监听器（开启/停止）。
- LightClient.VoiceOutputListener：语音上传后，TTS结果返回监听器，返回tts的url。
- LightClient.AudioPlayerListener: 语音上传后，音频结果返回监听器，返回音频的url。
- LightClient.IAudioInputListener：语音输入状态监听，分别有正在录音、录音结束、录音错误。
- CustomAudioRecord： 用户自定义的录音器，需要代码中自行调用上传语音，具体用法见接口说明。 
- LightduerResource： 数据点类，用户在开放平台创建的数据点可以通过此类定义。

### 3.2  接口说明

#### 3.2.1  SDK初始化

初始化SDK分两步完成：1.initSdk  2.connectServer

#### 3.2.1.1  init sdk

**接口功能描述**

初始化SDK,在自定义MainActivity中调用如下方法,保证在调用LightDuerSDK其他接口前完成初始化,此方法可以在主线中调用。

**前置条件** 

无

**传入参数**

无

**回调函数** 

无

**实例**

```
DuerLightOSSDK.getInstance().getClient().init();
```

#### 3.2.1.2 connectServer

**接口功能描述**

连接lightduer server，在初始化SDK后连接duer server,此方法可以在主线中调用。

**前置条件** 

初始化SDK（DuerLightOSSDK.getInstance().getClient().init()）后调用;

**传入参数**

|类型|名称|描述|
|-|-|-|
|String|profile|profile文件，需要在开放平台上申请获取，读取profile文件转为String|
|LightClient.IConnectStatusListener|iConnectStatusListener|请求回调|

**回调函数** 

回调参数LightClient.IConnectStatusListener将在回调方法onConnectStatus(LightduerEvent lightduerEvent)中传递,lightduerEvent.getEventid()为LightduerEventId.DUER_EVENT_STARTED表示服务已经启动，其他状态则为启动失败。

**实例**

```
DuerLightOSSDK.getInstance().getClient().connectServer(profileSB.toString(), iConnectStatusListener);
LightClient.IConnectStatusListener iConnectStatusListener = new LightClient.IConnectStatusListener() {
    @Override
    public void onConnectStatus(LightduerEvent lightduerEvent) {
        if (lightduerEvent.getEventid() == LightduerEventId.DUER_EVENT_STARTED) {
            Toast.makeText(LightDuerMainActivity.this, R.string.connect_success, Toast.LENGTH_LONG).show();
        } else {
            Toast.makeText(LightDuerMainActivity.this, R.string.connect_fail, Toast.LENGTH_LONG).show();
        }
    }
};
```

#### 3.2.2 设置并初始化监听器
#### 3.2.2.1 setAudioInputListener

**接口功能描述**

设置此监听器用于监听当前录音器的状态：开始录音、录音停止、录音错误

**前置条件** 

无

**传入参数**

|类型|名称|描述|
|-|-|-|
|LightClient.IAudioInputListener|IAudioInputListener|输入语音状态回调|

**回调函数** 

有三个回调函数：
1. onStartRecord 设备开始录音
2. onStopRecord 设备停止录音
3. onErrorRecord(int errorCode) 设备录音发生错误。errorCode目前只有ClientImpl.ERROR_AUDIO_NO_PERMISSION，此code表示打开录音器没有权限。

**实例**

```
DuerLightOSSDK.getInstance().getClient().setAudioInputListener(new LightClient.IAudioInputListener() {
    @Override
    public void onStartRecord() {
    }

    @Override
    public void onStopRecord() {
    }

    @Override
    public void onErrorRecord(int errorCode) {
        Log.d(TAG, "record error code = " + errorCode);
        if (ClientImpl.ERROR_AUDIO_NO_PERMISSION == errorCode) {
            ActivityCompat.requestPermissions(LightDuerMainActivity.this, new String[]{Manifest.permission.RECORD_AUDIO}, 1);
        }
    }
});
```

#### 3.2.2.2 setVoiceOutputListener

**接口功能描述**

此监听器用于获取语音上传后的tts返回，返回的数据为tts的url。

**前置条件** 

无

**传入参数**

|类型|名称|描述|
|-|-|-|
|LightClient.VoiceOutputListener|VoiceOutputListener|返回的tts结果回调|

**回调函数** 

回调函数为：speak(String url)，url为上传语音后返回tts的url地址。

**实例**

```
DuerLightOSSDK.getInstance().getClient().setVoiceOutputListener(new LightClient.VoiceOutputListener() {
    @Override
    public int speak(String url) {
        return 0;
    }
});
```

#### 3.2.2.3 setAudioPlayerListener

**接口功能描述**

此监听器用于获取语音上传后的音频数据的返回，返回的数据为音频的url。

**前置条件** 

无

**传入参数**

|类型|名称|描述|
|-|-|-|
|LightClient.AudioPlayerListener|AudioPlayerListener|返回音频结果回调|

**回调函数** 

目前支持的回调函数有：int play(String url) {}，此url为音频数据的url地址。其他回调函数暂时不支持。

**实例**

```
DuerLightOSSDK.getInstance().getClient().setAudioPlayerListener(new LightClient.AudioPlayerListener() {
    @Override
    public int play(String url) {
        resultTv.setText("(AudioPlayerListener)url = " + url);
        mMediaPlayer.playUrl(url, LightDuerMediaPlayer.TYPE_AUDIO);
        return 0;
    }

    @Override
    public int stop() {
        return 0;
    }

    @Override
    public int resume(String url, int offset) {
        return 0;
    }

    @Override
    public int pause() {
        return 0;
    }

    @Override
    public int getAudioPlayProgress() {
        return 0;
    }
});
```

#### 3.2.3 数据点相关接口

添加数据点，需要在开放平台上添加对应的数据点，可以用于控制设备。

#### addControlPoint

**接口功能描述**

添加数据点，在代码中创建出对应的数据点添加对数据点的监听。

**前置条件** 

无

**传入参数**

|类型|名称|描述|
|-|-|-|
|LightduerResource |resources|数据点|

LightduerResource详细说明：

```
public LightduerResource(int mode, int allowed, String path, byte[] staticData, LightduerResourceListener callback) {}
```

int mode：   数据点的类型动态(LightduerResource.DUER_RES_MODE_DYNAMIC)和静态(LightduerResource.DUER_RES_MODE_STATIC)两种类型。  
int allowed：有四种类型{DUER_RES_OP_GET, DUER_RES_OP_PUT, DUER_RES_OP_POST, DUER_RES_OP_DELETE}  
String path：   开放平台上设置的数据点名称  
byte[] staticData：动态数据点可以传null，静态数据点传入静态数据点string获取到的bytes即可（static_resource.getBytes()）  
LightduerResourceListener callback：数据点的监听器，内有一个监听函数

```
public int callback(LightduerContext context, LightduerMessage message, LightduerAddress address) {}
```

LightduerContext context： 设置数据点的上下文  
LightduerMessage message： 数据点中返回的内容，可以通过message.getPayload()获取  
LightduerAddress address： 区分message是从那个地址过来的

**回调函数** 

无

**实例**

```
DuerLightOSSDK.getInstance().getClient().addControlPoint(resources);
```

#### 3.2.4 Android通用Record（非CustomAudioRecord）相关接口
使用sdk封装好的一套录音上传语音数据完成语音交互。

#### 3.2.4.1 startRecord

**接口功能描述**

开始语音交互。

**前置条件** 

无

**传入参数**

无

**回调函数** 

无

**实例**

```
DuerLightOSSDK.getInstance().getClient().startRecord();
```

#### 3.2.5 CustomAudioRecord相关接口

此接口为用户自定义audiorecord来完成语音录制并上传，需要注意的是此接口完全使用另一套流程。不需要再调用DuerLightOSSDK.getInstance().getClient().startRecord();接口开始语音交互。

#### 3.2.5.1 getCustomAudioRecord

**接口功能描述**

获取一个自定义的播放器。

**前置条件** 

无

**传入参数**

无

**回调函数** 

无

**实例**

```
DuerLightOSSDK.getInstance().getClient().getCustomAudioRecord();
```

#### 3.2.5.2 getAudioStatus

**接口功能描述**

获取当前的audioRecorder的状态：停止状态、开始状态。

**前置条件** 

无

**传入参数**

无

**回调函数** 

无

**返回值** 

|类型|名称|描述|
|-|-|-|
|CustomAudioRecord.Status |status|CustomAudioRecord.Status.STARTED 开始状态；CustomAudioRecord.Status.STOPED 停止状态|

**实例**

```
mCustomAudioRecord.getAudioStatus();
```

#### 3.2.5.3 startRecord

**接口功能描述**

在调用传输接口先调用此接口来通知sdk接受语音数据。

**前置条件** 

无

**传入参数**

|类型|名称|描述|
|-|-|-|
|int|samplerate|采样率，目前支持16000|

**回调函数** 

无

**实例**

```
mCustomAudioRecord.startRecord(16000);
```
#### 3.2.5.4 sendVoiceData

**接口功能描述**

向sdk发送语音数据。

**前置条件** 

先调用mCustomAudioRecord.startRecord();通知sdk开始接受数据

**传入参数**

|类型|名称|描述|
|-|-|-|
|byte[]|voiceData|录制的语音数据|

**回调函数** 

无

**实例**
```
private void sendDataToCustomAudioData() {
    BufferedInputStream in = null;
    try {
        in = new BufferedInputStream(getAssets().open("weather.pcm"));
            int buf_size = 1024;
            byte[] buffer = new byte[buf_size];
            while (-1 != (in.read(buffer, 0, buf_size))) {
            DuerLightOSSDK.getInstance().getClient().getCustomAudioRecord().sendVoiceData(buffer);
        }
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        try {
            in.close();
            in2.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
();通知sdk开始接受数据

**传入参数**

|类型|名称|描述|
|-|-|-|
|byte[]|voiceData|录制的语音数据|

**回调函数** 

无

**实例**
```
private void sendDataToCustomAudioData() {
    BufferedInputStream in = null;
    try {
        in = new BufferedInputStream(getAssets().open("weather.pcm"));
            int buf_size = 1024;
            byte[] buffer = new byte[buf_size];
            while (-1 != (in.read(buffer, 0, buf_size))) {
            DuerLightOSSDK.getInstance().getClient().getCustomAudioRecord().sendVoiceData(buffer);
        }
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        try {
            in.close();
            in2.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
