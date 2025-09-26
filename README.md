# 快手Unidbg 13.8.40.44479 设备注册加签名流程文档

## 概述

本文档详细描述了快手Unidbg项目中设备注册（EGID生成和注册）的完整流程。该流程通过模拟Android环境，生成设备指纹信息，并向快手服务器注册获取设备标识符（EGID）。

## 核心组件

## 设备注册流程详解



### 阶段1：环境初始化

#### 1.1 Unidbg模拟器创建
```java
// 创建32位Android模拟器
emulator = AndroidEmulatorBuilder.for32Bit()
    .setProcessName("com.smile.gifmaker")
    .build();

// 设置系统类库解析器
memory.setLibraryResolver(new AndroidResolver(23));

// 创建Dalvik虚拟机
vm = emulator.createDalvikVM(apkFile);
```

#### 1.2 库加载和JNI设置
```java
// 加载目标库文件
DalvikModule dm_kwsgmain = vm.loadLibrary("kwsgmain", true);

// 设置JNI回调
vm.setJni(this);

// 调用JNI OnLoad
dm_kwsgmain.callJNI_OnLoad(emulator);
```

#### 1.3 初始化doCommandNative
```java
// 第一次执行doCommandNative，初始化环境
Object[] objects1 = new Object[]{"", Android.ANDRIOD_UUID1, "", "", APP_Context, "", ""};
doCommandNative(10412, objects1);
```

### 阶段2：设备参数构建

#### 2.1 设备号生成策略

在Android环境中，当无法获取到真实设备号时，系统会生成随机设备号来保证设备注册流程的正常进行。

##### 随机设备号生成方法
当Android获取不到设备号时，系统会按以下顺序处理：
首先尝试获取系统Android ID - 通过AndroidIdInterceptor.getString()获取
如果系统Android ID无效或为空 - 调用m61483b()方法生成随机设备ID
随机设备ID生成逻辑：
使用SecureRandom().nextInt()生成随机数
转换为16位十六进制字符串
格式化为ANDROID_xxxxxxxxxxxxxxxx格式
保存到SharedPreferences的new_random_android_id键中
持久化存储 - 同时保存到SharedPreferences和文件中，确保下次启动时能读取到相同的设备ID
```

##### 设备号生成逻辑
```java
/* renamed from: b */
public static String m61483b() {
    String strM112248z = null;
    try {
        // 使用SecureRandom生成16位十六进制随机数
        strM112248z = TextUtils.m112248z(Long.toHexString(new SecureRandom().nextInt()), 16, '0');
        C5396b.m11649h().mo122047i("DeviceIdManager", "generateRandomDeviceId generatedRandomId = " + strM112248z, new Object[0]);
        if (!TextUtils.m112246x(strM112248z)) {
            // 保存到SharedPreferences
            f108268f.edit().putString("new_random_android_id", strM112248z).apply();
        }
    } catch (Throwable th3) {
        C5396b.m11649h().mo122047i("DeviceIdManager", "generateRandomDeviceId exception = " + th3.getMessage(), new Object[0]);
    }
    return "ANDROID_" + strM112248z;  // 返回格式：ANDROID_xxxxxxxxxxxxxxxx
}
```

#### 2.2 EgidNano参数设置
系统构建包含119个字段的设备参数对象：

```java
EgidNano egidNano = new EgidNano();
// 设置关键设备参数
egidNano.a5 = deviceId;                          // 设备ID（真实设备号或随机生成的设备号）
egidNano.a7 = Android.DID;                       // 设备标识符
egidNano.a16 = "android-build";                 // 构建类型
egidNano.a19 = "******";                        // 硬件平台
egidNano.a22 = Android.KUAISHOU_VERSION;         // 快手版本
egidNano.a23 = Android.Manufacturer;             // 制造商
egidNano.a27 = Android.ANDROID_POP;              // Android版本
egidNano.a28 = Android.Hardware_Name;            // 硬件名称
egidNano.a29 = Android.Android_Dalvik;           // Dalvik版本
egidNano.a30 = Android.Android_Version;           // 系统版本
egidNano.a34 = Android.Width_Height;             // 屏幕分辨率
egidNano.a35 = Android.ANDROID_VERSION;          // Android版本号
egidNano.a40 = Android.Fingerprint;              // 设备指纹
egidNano.a52 = Android.Android_Type;             // Android类型
egidNano.a58 = Android.Android_Type;             // Android类型
egidNano.a61 = Android.Manufacturer;             // 制造商
egidNano.a67 = "cn";                            // 国家代码
egidNano.a84 = UUID.randomUUID().toString().replace("-", ""); // 动态UUID
egidNano.a89 = String.format("%09d", new Random().nextInt(1000000000)); // 9位随机数
egidNano.a92 = String.valueOf(System.currentTimeMillis()); // 当前时间戳
```

#### 2.2 CRC32校验码计算
```java
// 计算所有字段的CRC32校验码
String[] fields = {egidNano.a1, egidNano.a2, ..., egidNano.a119};
CRC32 crc32 = new CRC32();
for (int i = 0; i < fields.length; i++) {
    crc32.update(fields[i].getBytes());
}
egidNano.a14 = "AND:" + crc32.getValue();
```

### 阶段3：DeviceInfo生成

#### 3.1 调用doCommandNative(10400)
```java
// 构造参数
byte[] egidNano2 = Egid.egidarg1();
Object[] objects2 = new Object[]{egidNano2, Android.ANDRIOD_UUID1, 0, "", APP_Context, true, true, Android.ANDRIOD_UUID2};

// 调用native方法获取deviceInfo
ByteArray byteArray = (ByteArray) doCommandNative(10400, objects2);
byte[] bytes = byteArray.getValue();

// Base64编码并URL编码
deviceInfo = URLEncoder.encode(Encryption.Base64_MimeEncrypt(bytes));
```

### 阶段4：Sign签名生成

#### 4.1 调用doCommandNative(10418)
```java
// 构造签名参数
String[] stringArray = new String[]{"KUAISHOU" + Egid.toTime + "2" + deviceInfo.trim()};
Object[] objects3 = new Object[]{stringArray, Android.ANDRIOD_UUID1, -1, false, APP_Context, "", true, Android.ANDRIOD_UUID2};

// 获取签名
sign = doCommandNative(10418, objects3).toString().replace("\"", "");
```

### 阶段5：EGID请求和注册

#### 5.1 构建EGID请求体
```java
RequestBody egidBody = new FormBody.Builder()
    .add("productName", "KUAISHOU")
    .add("ts", String.valueOf(toTime))
    .add("deviceInfo", deviceInfo)
    .add("sign", sign)
    .add("sv", "2")
    .add("rdid", rdid)
    .add("didtag", "-1")
    .build();
```

#### 5.2 发送EGID注册请求
```java
// 随机选择服务器地址
String primaryUrl = "https://gdfpsec.gifshow.com/rest/infra/gdfp/report/kuaishou/android";
String fallbackUrl = "https://gdfpsec.ksapisrv.com/rest/infra/gdfp/report/kuaishou/android";
String chosenUrl = ThreadLocalRandom.current().nextBoolean() ? primaryUrl : fallbackUrl;

// 发送POST请求
Request request = new Request.Builder()
    .url(chosenUrl)
    .addHeader("Host", hostHeader)
    .addHeader("User-Agent", "okhttp/3.12.13")
    .addHeader("Content-Type", "application/x-www-form-urlencoded")
    .addHeader("Cookie", "did=" + Android.DID)
    .post(egidBody)
    .build();

Response response = client.newCall(request).execute();
```

#### 5.3 解析EGID响应
```java
JSONObject jsonObject = JSONObject.parseObject(resultBody);
String egid = jsonObject.get("egid").toString();
```

## 设备号生成策略说明

### 背景
在Unidbg模拟的Android环境中，由于无法访问真实的硬件设备信息，系统无法获取到真实的设备号（如IMEI、Android ID等）。为了保证设备注册流程的正常进行，需要生成符合格式要求的随机设备号。

### 生成策略
1. **优先获取真实设备号**：首先尝试从Android系统获取真实的设备标识符
2. **随机设备号生成**：当无法获取真实设备号时，使用`generateRandomHex()`方法生成随机十六进制字符串
3. **格式要求**：生成的设备号必须为偶数长度的十六进制字符串，通常为32位
4. **安全性**：使用`SecureRandom`确保生成的随机数具有足够的随机性和安全性

### 应用场景
- Unidbg模拟器环境
- 测试环境
- 无法获取真实设备信息的Android环境
- 需要保证设备注册流程稳定性的场景

### 注意事项
- 生成的随机设备号在单次会话中保持一致
- 不同会话会生成不同的随机设备号
- 随机设备号需要符合快手服务器的格式要求
- 建议在日志中记录设备号的生成情况，便于调试和监控


**免责声明**: 本项目仅用于安全研究和学习目的，请勿用于非法用途。使用者需自行承担相关责任。



![6163523eb9b54ba256422d8f2c4ff0d6](https://github.com/user-attachments/assets/24a80507-3b18-45ea-bf9d-1383d30ad247)
