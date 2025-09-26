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
- 支持日千万级别爬虫算法

curl -X POST 'https://az1-api-js.gifshow.com/rest/n/tube/serialPageData?appver=13.8.40.44479&ver=13.8&oDid=ANDROID_4ede1a12073cfb21&did_gt=1758918144821&cold_launch_time_ms=1758918144821&rdid=ANDROID_7a2846ce0a685b33&did=ANDROID_3e9f76e5f40d3609&earphoneMode=1&mod=Google%28Pixel%203%20XL%29&isp=CMCC&language=zh-cn&ud=56768391&did_tag=0&egid=DFP64B4758BC96F746FDB161B3AE60AC9731BBFAAB31B5A8EB134BBC88EED1AA&thermal=10000&net=WIFI&kcv=1598&app=0&kpf=ANDROID_PHONE&bottom_navigation=false&android_os=0&boardPlatform=sdm845&kpn=KUAISHOU&newOc=GENERIC&androidApiLevel=31&slh=0&country_code=cn&nbh=168&hotfix_ver&keyconfig_state=2&cdid_tag=2&sys=ANDROID_12&max_memory=256&oc=GENERIC&sh=2960&deviceBit=0&browseType=4&ddpi=560&socName=Qualcomm%20Snapdragon%20845&is_background=0&c=GENERIC&sw=1440&ftt&abi=arm64&userRecoBit=0&device_abi=arm64&icaver=1&totalMemory=3579&grant_browse_type=AUTHORIZED&iuid&sbh=171&darkMode=false&sig=7ef1367f6f17a46aad9a20e473dfcdab&__NS_sig3=7b6a1f395b3a87783333303179311fd42addfe412e222c3a&__NS_xfalcon=72c0631ac9093731221ed9ea34b460ddf1bfce1ffbd47ee5f9c1a7ec025f7025&__NStokensig=8bac832a1e2ea6669f2c4d395111a953d2573c43a4d5ed69265a39cb3a0a09a1' -H 'Content-Type: application/x-www-form-urlencoded' --data-urlencode 'cs=false' --data-urlencode 'client_key=3c2cd3f3' --data-urlencode 'videoModelCrowdTag=1_10' --data-urlencode 'os=android' --data-urlencode 'count=20' --data-urlencode 'kuaishou.api_st=Cg9rdWFpc2hvdS5hcGkuc3QSoAH32qdY9EQfhXIY6tH5UFRiDDjjsktvk-4TOnHmEcmjBRDvit6nS-xM-dtg_eLKfrTad_9YozjDzeiv5V1AQgVjOe1UJAQawFu5xHlwzhPmdj_SNCMXXq-EMzbE_AoVcnWjSz_1sv6p2YpW2Cp9dfj-4h0nK-cGryiPwblH1Dii07CCYU1Vs0-lYyju56AW916-_OyL_YgzridzWkT53MPOGhLmHaQkwpVHXJtSDkMe1YHGH_QiIL2iNCw08FTFLnXC_HkSSs8WES7logXxg1OybQfb4ml5KAUwAQ' --data-urlencode 'uQaTag=1##swLdgl:99#ecPp:8-#cmNt:11' --data-urlencode 'profileUserId=1087835257' --data-urlencode 'token=18fe8e68e27e49d987e8d933e20cf53f-56768391'


{
  "result": 1,
  "pcursor": "20",
  "host-name": "public-bjx-c34-kce-node2347.idchb1az1.hb1.kwaidc.com",
  "tubeInfo": [
    {
      "collected": false,
      "serialType": 1,
      "firstEpisode": {
        "photoCdnUrl": [
          {
            "cdn": "hw2.a.kwimgs.com",
            "url": "http://hw2.a.kwimgs.com/upic/2025/09/26/14/BMjAyNTA5MjYxNDU1MzFfMTA4NzgzNTI1N18xNzU4MjkyODEwOTZfMF8z_Be4c172653721684fad78b88bdb44efca.jpg?tag=1-1758918181-unknown-0-qh07oncdz3-1b33ae2f88eff45f&clientCacheKey=3xhvkdq62tfcrji.jpg&di=JAmKACWhY6BZDt89hZaTVw==&bp=10001"
          }
        ],
        "linkUrl": "kwai://episode/play?serialId=13056813&serialType=1&photoId=5200813317425190714&selectedPhotoId=5200813317425190714&sourcePhotoPage=attab",
        "name": "第1集",
        "id": 6263399441,
        "photoId": 175829281096,
        "tubeId": 13056813
      },
      "disableViewCountByFilm": false,
      "likeCount": 0,
      "name": "无限深渊列车1-极限逃亡",
      "id": "5x7p8q6xx4rg2cs",
      "finished": true,
      "bizType": 10,
      "totalEpisodesCount": 40,
      "extParams": "{\"display\":{\"poster\":\"\",\"director\":\"\",\"producer\":\"\",\"mainActors\":\"\",\"companyName\":\"\",\"firstPlayDay\":0,\"scriptwriter\":\"\",\"episodeLength\":0,\"copyrightNumber\":\"\",\"institutionName\":\"\",\"producerCompany\":\"\",\"visibleToOutside\":false,\"coProducerCompany\":\"\",\"onlinePlayPlatform\":\"\",\"superviseTheManufacture\":\"\"},\"keyword\":\"无限深渊列车1-极限逃亡\",\"channelIds\":[],\"screenType\":0,\"contentType\":0,\"description\":\"剧情简介：一辆行驶在荒野的列车突然停车，乘客们被拉进一个微信群，全员禁言。一个匿名者开始发言：「……希望各位乘客能够认真阅读本群提示，将生存守则牢记于心。注意，一定要牢记……」在这个危险的深夜，所有不遵循匿名者提示的人，都消失了。恐惧使我不惜一切按匿名者的提示行事，哪怕面对发小的消失，我也保持沉默，直到最后，我终于取得本车厢唯一的生机。\",\"orderWeight\":0,\"backgroundTags\":[],\"tubeSourceType\":3,\"firstOnlineTime\":1758870111134,\"tubeAdoptionStatus\":1,\"tube_biz_type\":\"10\"}",
      "viewCount": 10199,
      "adoptionType": 0,
      "totalEpisodesCountIgnoreStatus": 40
    },
    {
      "collected": false,
      "serialType": 1,
      "firstEpisode": {
        "photoCdnUrl": [
          {
            "cdn": "hw2.a.kwimgs.com",
            "url": "http://hw2.a.kwimgs.com/upic/2025/09/26/12/BMjAyNTA5MjYxMjExMTBfMTA4NzgzNTI1N18xNzU4MjAzNjIxNTJfMF8z_Bfad466a6996697d59272a35ac21ef73c.jpg?tag=1-1758918181-unknown-0-pmgnhouvqc-46e2b3ac0092b357&clientCacheKey=3x4xui592j2848u.jpg&di=JAmKACWhY6BZDt89hZaTVw==&bp=10001"
          }
        ],
        "linkUrl": "kwai://episode/play?serialId=13054152&serialType=1&photoId=5241908667355171308&selectedPhotoId=5241908667355171308&sourcePhotoPage=attab",
        "name": "第1集",
        "id": 6258533053,
        "photoId": 175820362152,
        "tubeId": 13054152
      },
      "disableViewCountByFilm": false,
      "likeCount": 0,
      "name": "爹地别慌，小锦鲤专治不服",
      "id": "5xe8utp86unu9ta",
      "finished": true,
      "bizType": 10,
      "totalEpisodesCount": 66,
      "extParams": "{\"display\":{\"poster\":\"\",\"director\":\"\",\"producer\":\"\",\"mainActors\":\"\",\"companyName\":\"\",\"firstPlayDay\":0,\"scriptwriter\":\"\",\"episodeLength\":0,\"copyrightNumber\":\"\",\"institutionName\":\"\",\"producerCompany\":\"\",\"visibleToOutside\":false,\"coProducerCompany\":\"\",\"onlinePlayPlatform\":\"\",\"superviseTheManufacture\":\"\"},\"keyword\":\"爹地别慌，小锦鲤专治不服\",\"channelIds\":[],\"screenType\":0,\"contentType\":0,\"description\":\"嘟嘟本是海城首富萧家唯一嫡孙女，出生时因大师预言她八字过硬，与萧家相克，最终被送到乡下养了五年。实则嘟嘟拥有不为人知的锦鲤体质，不仅让残废五年的萧凛夜重新康复，还能听到万物心声，帮助自己和小叔躲过重重危机……嘟嘟好不容易回到萧家，却发现亲生父母早已有了养女，对自己百般嫌弃。好在小叔萧凛夜坚信嘟嘟是福星，愿意将她过继到名下做女儿。从此，嘟嘟凭借锦鲤体质，带着“亲选爹地”步步高升。\",\"orderWeight\":0,\"backgroundTags\":[],\"tubeSourceType\":3,\"firstOnlineTime\":1758859984345,\"tubeAdoptionStatus\":1,\"tube_biz_type\":\"10\"}",
      "viewCount": 491208,
      "adoptionType": 0,
      "totalEpisodesCountIgnoreStatus": 66
    },
    {
      "collected": false,
      "serialType": 1,
      "firstEpisode": {
        "photoCdnUrl": [
          {
            "cdn": "ali2.a.kwimgs.com",
            "url": "http://ali2.a.kwimgs.com/upic/2025/09/26/12/BMjAyNTA5MjYxMjA2MDNfMTA4NzgzNTI1N18xNzU4MjAwNjM1MTRfMF8z_Beec68d7a7fc78a1eddb0c2a123b654d2.jpg?tag=1-1758918181-unknown-0-q9zg3xhgkq-c659f5f5e0bc29c6&clientCacheKey=3xc68c35m5e95xe.jpg&di=JAmKACWhY6BZDt89hZaTVw==&bp=10001"
          }
        ],
        "linkUrl": "kwai://episode/play?serialId=13054082&serialType=1&photoId=5229523767565032282&selectedPhotoId=5229523767565032282&sourcePhotoPage=attab",
        "name": "第1集",
        "id": 6258424052,
        "photoId": 175820063514,
        "tubeId": 13054082
      },
      "disableViewCountByFilm": false,
      "likeCount": 0,
      "name": "梨花枪影破山河",
      "id": "5x4ixvh94brtqk2",
      "finished": true,
      "bizType": 10,
      "totalEpisodesCount": 60,
      "extParams": "{\"display\":{\"poster\":\"\",\"director\":\"\",\"producer\":\"\",\"mainActors\":\"\",\"companyName\":\"\",\"firstPlayDay\":0,\"scriptwriter\":\"\",\"episodeLength\":0,\"copyrightNumber\":\"\",\"institutionName\":\"\",\"producerCompany\":\"\",\"visibleToOutside\":false,\"coProducerCompany\":\"\",\"onlinePlayPlatform\":\"\",\"superviseTheManufacture\":\"\"},\"keyword\":\"梨花枪影破山河\",\"channelIds\":[],\"screenType\":0,\"contentType\":0,\"description\":\"九凤将星高凤兰为履行亡夫遗愿，封印修为隐居青云武馆18年。其子高天问不知母亲身份，因父亲之死立志加入巡捕司复仇，却遭高凤兰阻拦。母子激烈冲突之际，金龙会余孽王大海上门挑衅，欲夺武馆巡捕司名额。高凤兰为救子冲破封印，暴雨梨花枪重现江湖，身份初露端倪。高凤兰重训武馆弟子备战将星选拔。选拔赛中，青岚操控赛制纵容金龙会残害武者，更故意安排高天问与季如烟同室操戈。季如烟中毒重伤高天问，生死关头高天问觉醒凤血圣体反败为胜。青岚公然剥夺高天问资格，高凤兰蒙面现身，揭露其勾结金龙会、害死徐修远的真相。高凤兰以九凤之姿重归，三大战将臣服。为清算血仇，她蒙眼三招败青岚，将其罪行公之于众。高天问以单手持枪对决青岚，手刃杀父仇人。金龙会阴谋溃败，巡捕司拨乱反正。\",\"orderWeight\":0,\"backgroundTags\":[],\"tubeSourceType\":3,\"firstOnlineTime\":1758866348476,\"tubeAdoptionStatus\":1,\"tube_biz_type\":\"10\"}",
      "viewCount": 10737,
      "adoptionType": 0,
      "totalEpisodesCountIgnoreStatus": 60
    },
    {
      "collected": false,
      "serialType": 1,
      "firstEpisode": {
        "photoCdnUrl": [
          {
            "cdn": "hw2.a.kwimgs.com",
            "url": "http://hw2.a.kwimgs.com/upic/2025/09/25/16/BMjAyNTA5MjUxNjE1MTNfMTA4NzgzNTI1N18xNzU3NjYyMjk2MTRfMF8z_B14ff8622e57459a34a3aa1c805e94404.jpg?tag=1-1758918181-unknown-0-nqntjc5dfw-145b2660ad2f5526&clientCacheKey=3xhnjvim9jtmi2u.jpg&di=JAmKACWhY6BZDt89hZaTVw==&bp=10001"
          }
        ],
        "linkUrl": "kwai://episode/play?serialId=13034816&serialType=1&photoId=5247256689690009528&selectedPhotoId=5247256689690009528&sourcePhotoPage=attab",
        "name": "第1集",
        "id": 6220862066,
        "photoId": 175766229614,
        "tubeId": 13034816
      },
      "disableViewCountByFilm": false,
      "likeCount": 0,
      "name": "你赐我满城风雨",
      "id": "5xs8dkarwghgeva",
      "finished": true,
      "bizType": 10,
      "totalEpisodesCount": 83,
      "extParams": "{\"display\":{\"poster\":\"\",\"director\":\"\",\"producer\":\"\",\"mainActors\":\"\",\"companyName\":\"\",\"firstPlayDay\":0,\"scriptwriter\":\"\",\"episodeLength\":0,\"copyrightNumber\":\"\",\"institutionName\":\"\",\"producerCompany\":\"\",\"visibleToOutside\":false,\"coProducerCompany\":\"\",\"onlinePlayPlatform\":\"\",\"superviseTheManufacture\":\"\"},\"keyword\":\"你赐我满城风雨\",\"channelIds\":[],\"screenType\":0,\"contentType\":0,\"description\":\"五年前，江雾亲手将沈司烬送入监狱，五年后，他权势滔天归来，对她恨之入骨，摧毁了她残存的生活，断她一切生路，直到乐乐重病垂危，血缘关系撕碎了所有谎言，身陷绝境的她，在偏执旧爱与温柔守护间，面临最终抉择。\",\"orderWeight\":0,\"backgroundTags\":[],\"tubeSourceType\":3,\"firstOnlineTime\":1758788903381,\"tubeAdoptionStatus\":1,\"tube_biz_type\":\"10\"}",
      "viewCount": 55856,
      "adoptionType": 0,
      "totalEpisodesCountIgnoreStatus": 83
    },
    {
      "collected": false,
      "serialType": 1,
      "firstEpisode": {
        "photoCdnUrl": [
          {
            "cdn": "ali2.a.kwimgs.com",
            "url": "http://ali2.a.kwimgs.com/upic/2025/09/24/15/BMjAyNTA5MjQxNTU3MzFfMTA4NzgzNTI1N18xNzU2OTM1MDQ5NDRfMF8z_Ba96dd5bdae48304912b7344515f6f4b4.jpg?tag=1-1758918181-unknown-0-q8eeporve5-a9e76cffb84d8503&clientCacheKey=3x5m8rmaezatn5w.jpg&di=JAmKACWhY6BZDt89hZaTVw==&bp=10001"
          }
        ],
        "linkUrl": "kwai://episode/play?serialId=13008069&serialType=1&photoId=5195465293488613929&selectedPhotoId=5195465293488613929&sourcePhotoPage=attab",
        "name": "第1集",
        "id": 6177588764,
        "photoId": 175693504944,
        "tubeId": 13008069
      },
      "disableViewCountByFilm": false,
      "likeCount": 0,
      "name": "朝朝待祈晏",
      "id": "5x7bcvx6zcn3xwu",
      "finished": true,
      "bizType": 10,
      "totalEpisodesCount": 60,
      "extParams": "{\"display\":{\"poster\":\"\",\"director\":\"\",\"producer\":\"\",\"mainActors\":\"\",\"companyName\":\"\",\"firstPlayDay\":0,\"scriptwriter\":\"\",\"episodeLength\":0,\"copyrightNumber\":\"\",\"institutionName\":\"\",\"producerCompany\":\"\",\"visibleToOutside\":false,\"coProducerCompany\":\"\",\"onlinePlayPlatform\":\"\",\"superviseTheManufacture\":\"\"},\"keyword\":\"朝朝待祈晏\",\"channelIds\":[],\"screenType\":0,\"contentType\":0,\"description\":\"\\\"剧情简介：前世被嫡姐、嫡母等人联手迫害惨死的宋朝朝，重生回到了要给嫡姐和世子当试睡丫头的那天。她决定这一世要借助宣平侯萧祁晏的势力，来完成复仇。她知道萧祁晏会在宫宴上被长公主算计下药，意识朦胧地回府，所以有预谋地撞上他，做了他的解药。第二日她精准拿捏了萧祁晏的愧疚心理，使萧祁晏承诺会对她负责，并给了她信物。事后，宋宝珠得知宋朝朝忤逆了她的命令，并没有去当试睡丫头。宋宝珠骂她给脸不要脸，让人把她溺到池塘里教训，却不小心直接把宋朝朝溺死，惊慌之下，找母亲一起草草把她的“尸体”丢到乱葬岗。（假的，女主诈死）宋朝朝与自己的丫鬟配合，诈死脱身，改从母姓为顾，将信物和自己的生辰八字一起送到了萧祁晏手上，要萧祁晏信守承诺。没多久，宣平侯忽然娶妻，满朝震惊，只因他娶的是个淮州的小商户之女。\",\"orderWeight\":0,\"backgroundTags\":[],\"tubeSourceType\":3,\"firstOnlineTime\":1758704638932,\"tubeAdoptionStatus\":1,\"tube_biz_type\":\"10\"}",
      "viewCount": 31452,
      "adoptionType": 0,
      "totalEpisodesCountIgnoreStatus": 60
    },
    {
      "collected": false,
      "serialType": 1,
      "firstEpisode": {
        "photoCdnUrl": [
          {
            "cdn": "ali2.a.kwimgs.com",
            "url": "http://ali2.a.kwimgs.com/upic/2025/09/24/15/BMjAyNTA5MjQxNTQ2MzJfMTA4NzgzNTI1N18xNzU2OTI4NzU1NTlfMF8z_Bad076d445774b841a07720325bfe2832.jpg?tag=1-1758918181-unknown-0-zarkburamr-999b1fe75b390389&clientCacheKey=3xzwmfpftykzx52.jpg&di=JAmKACWhY6BZDt89hZaTVw==&bp=10001"
          }
        ],
        "linkUrl": "kwai://episode/play?serialId=13007704&serialType=1&photoId=5204753967115659539&selectedPhotoId=5204753967115659539&sourcePhotoPage=attab",
        "name": "第1集",
        "id": 6177320358,
        "photoId": 175692875559,
        "tubeId": 13007704
      },
      "disableViewCountByFilm": false,
      "likeCount": 0,
      "name": "云缕诗心",
      "id": "5xdduxsk8rb5xry",
      "finished": true,
      "bizType": 10,
      "totalEpisodesCount": 82,
      "extParams": "{\"display\":{\"poster\":\"\",\"director\":\"\",\"producer\":\"\",\"mainActors\":\"\",\"companyName\":\"\",\"firstPlayDay\":0,\"scriptwriter\":\"\",\"episodeLength\":0,\"copyrightNumber\":\"\",\"institutionName\":\"\",\"producerCompany\":\"\",\"visibleToOutside\":false,\"coProducerCompany\":\"\",\"onlinePlayPlatform\":\"\",\"superviseTheManufacture\":\"\"},\"keyword\":\"云缕诗心\",\"channelIds\":[],\"screenType\":0,\"contentType\":0,\"description\":\"云雾缭绕间，正欲冲破那剑神桎梏的关键时刻，背后突来的凛冽杀意如冰锥扎入。未及反应，磅礴灵力瞬间溃散，眨眼便从云端跌落，沦为低阶武者。可心中剑意怎会熄灭，重握那三尺青锋，眸中满是决然，定要凭此剑，铸就那万古不朽的无敌威名。\",\"orderWeight\":0,\"backgroundTags\":[],\"tubeSourceType\":3,\"firstOnlineTime\":1758727313146,\"tubeAdoptionStatus\":1,\"tube_biz_type\":\"10\"}",
      "viewCount": 22115,
      "adoptionType": 0,
      "totalEpisodesCountIgnoreStatus": 82
    },
    {
      "collected": false,
      "serialType": 1,
      "firstEpisode": {
        "photoCdnUrl": [
          {
            "cdn": "ali2.a.kwimgs.com",
            "url": "http://ali2.a.kwimgs.com/upic/2025/09/24/10/BMjAyNTA5MjQxMDU5MThfMTA4NzgzNTI1N18xNzU2NzY5Mzk1NTVfMF8z_Bd335a003baae413efcd3cc8ac555262e.jpg?tag=1-1758918181-unknown-0-nk3m9coqxm-b33ca3d5c8bf298a&clientCacheKey=3x6e79btjt9t8uy.jpg&di=JAmKACWhY6BZDt89hZaTVw==&bp=10001"
          }
        ],
        "linkUrl": "kwai://episode/play?serialId=13002537&serialType=1&photoId=5192087592828701672&selectedPhotoId=5192087592828701672&sourcePhotoPage=attab",
        "name": "第1集",
        "id": 6166770201,
        "photoId": 175676939555,
        "tubeId": 13002537
      },
      "disableViewCountByFilm": false,
      "likeCount": 0,
      "name": "福星公主心声护国",
      "id": "5x6ifpqampy3d8y",
      "finished": true,
      "bizType": 10,
      "totalEpisodesCount": 46,
      "extParams": "{\"display\":{\"poster\":\"\",\"director\":\"\",\"producer\":\"\",\"mainActors\":\"\",\"companyName\":\"\",\"firstPlayDay\":0,\"scriptwriter\":\"\",\"episodeLength\":0,\"copyrightNumber\":\"\",\"institutionName\":\"\",\"producerCompany\":\"\",\"visibleToOutside\":false,\"coProducerCompany\":\"\",\"onlinePlayPlatform\":\"\",\"superviseTheManufacture\":\"\"},\"keyword\":\"福星公主心声护国\",\"channelIds\":[],\"screenType\":0,\"contentType\":0,\"description\":\"现代历史系大学生黎念念穿越到虞朝，成为太子之女，还拥有被他人读取心声的能力。她凭借对历史的了解，提前预警危机，助皇帝化解粮荒、识破毒丹阴谋，获封“灵昭佑圣公主”。她努力改变亲人命运，修复皇帝与太子关系，唤醒瑞王远离权力争斗。贵妃因听到黎念念心声动摇，关键时保护她实现自我救赎。景王发动宫变失败自刎。事后，皇帝加封黎念念为“镇国公主”，大虞避开灭亡危机，她也陷入被家人“争抢”的甜蜜烦恼。\",\"orderWeight\":0,\"backgroundTags\":[],\"tubeSourceType\":3,\"firstOnlineTime\":1758684091377,\"tubeAdoptionStatus\":1,\"tube_biz_type\":\"10\"}",
      "viewCount": 8317617,
      "adoptionType": 0,
      "totalEpisodesCountIgnoreStatus": 46
    },
    {
      "collected": false,
      "serialType": 1,
      "firstEpisode": {
        "photoCdnUrl": [
          {
            "cdn": "ali2.a.kwimgs.com",
            "url": "http://ali2.a.kwimgs.com/upic/2025/09/23/17/BMjAyNTA5MjMxNzU4MzZfMTA4NzgzNTI1N18xNzU2MzIyODk4NThfMF8z_B8732a50351294e02d7bf9b23bc05cd58.jpg?tag=1-1758918181-unknown-0-ijnztub9xe-bf6ca1959394cba1&clientCacheKey=3x6rdqzgwcnrj89.jpg&di=JAmKACWhY6BZDt89hZaTVw==&bp=10001"
          }
        ],
        "linkUrl": "kwai://episode/play?serialId=12990439&serialType=1&photoId=5235716217506957867&selectedPhotoId=5235716217506957867&sourcePhotoPage=attab",
        "name": "第1集",
        "id": 6134902443,
        "photoId": 175632289858,
        "tubeId": 12990439
      },
      "disableViewCountByFilm": false,
      "likeCount": 0,
      "name": "剑影天武风",
      "id": "5xg62ecvpkqzkky",
      "finished": true,
      "bizType": 10,
      "totalEpisodesCount": 92,
      "extParams": "{\"display\":{\"poster\":\"\",\"director\":\"\",\"producer\":\"\",\"mainActors\":\"\",\"companyName\":\"\",\"firstPlayDay\":0,\"scriptwriter\":\"\",\"episodeLength\":0,\"copyrightNumber\":\"\",\"institutionName\":\"\",\"producerCompany\":\"\",\"visibleToOutside\":false,\"coProducerCompany\":\"\",\"onlinePlayPlatform\":\"\",\"superviseTheManufacture\":\"\"},\"keyword\":\"剑影天武风\",\"channelIds\":[],\"screenType\":0,\"contentType\":0,\"description\":\"江晨作为中原天武阁独传弟子，受阁主慕容尚亲传，拥有龙国最强武功。下山时，师娘沈凌云告知师父为守护九转龙神诀而死，还将半本诀法传给江晨，称另一半在海城崔家小姐处，且师父早为江晨订下婚约。江晨赴约却遇崔家与陈家订婚并被羞辱，此时许江月上门求亲。江晨随其回家后得知，真实九转龙神诀纹在许江月背上，崔家骗婚，江晨为得诀法娶了许江月，后助其成海城最强财团，终修成正果。\",\"orderWeight\":0,\"backgroundTags\":[],\"tubeSourceType\":3,\"firstOnlineTime\":1758682717787,\"tubeAdoptionStatus\":1,\"tube_biz_type\":\"10\"}",
      "viewCount": 9256,
      "adoptionType": 0,
      "totalEpisodesCountIgnoreStatus": 92
    },
    {
      "collected": false,
      "serialType": 1,
      "firstEpisode": {
        "photoCdnUrl": [
          {
            "cdn": "ali2.a.kwimgs.com",
            "url": "http://ali2.a.kwimgs.com/upic/2025/09/23/17/BMjAyNTA5MjMxNzMzMzlfMTA4NzgzNTI1N18xNzU2MzA1NDg1MTBfMF8z_B66cc495c9a92cbc87399da5c8790d6c1.jpg?tag=1-1758918181-unknown-0-7wobdnqrmv-21fe67b6fc5be676&clientCacheKey=3xsck5t5jnu437k.jpg&di=JAmKACWhY6BZDt89hZaTVw==&bp=10001"
          }
        ],
        "linkUrl": "kwai://episode/play?serialId=12988811&serialType=1&photoId=5225583116809805279&selectedPhotoId=5225583116809805279&sourcePhotoPage=attab",
        "name": "第1集",
        "id": 6133950727,
        "photoId": 175630548510,
        "tubeId": 12988811
      },
      "disableViewCountByFilm": false,
      "likeCount": 0,
      "name": "太太赴约时",
      "id": "5xuyy6m84wta7ta",
      "finished": true,
      "bizType": 10,
      "totalEpisodesCount": 99,
      "extParams": "{\"display\":{\"poster\":\"\",\"director\":\"\",\"producer\":\"\",\"mainActors\":\"\",\"companyName\":\"\",\"firstPlayDay\":0,\"scriptwriter\":\"\",\"episodeLength\":0,\"copyrightNumber\":\"\",\"institutionName\":\"\",\"producerCompany\":\"\",\"visibleToOutside\":false,\"coProducerCompany\":\"\",\"onlinePlayPlatform\":\"\",\"superviseTheManufacture\":\"\"},\"keyword\":\"太太赴约时\",\"channelIds\":[],\"screenType\":0,\"contentType\":0,\"description\":\"\\\"沈晚瓷，作为沈家的大小姐，原本享受着优渥的生活，然而家族的突然倒台让她陷入了困境。为了家族的复兴，她被迫与薄氏集团的总裁薄荆舟达成结婚协议。\\n薄荆舟，作为商业巨头，他的心中也有着自己的情感和纠葛。他与大舞蹈家简唯宁的关系被沈晚瓷误会，导致了两人之间的裂痕。同时，他也对沈晚瓷与聂家大少聂煜城的关系心存疑虑，认为两人还藕断丝连。这种相互的误解和猜疑，使得他们的婚姻陷入了危机。\\n然而，在得知一切只是误会后，薄荆舟决定重新追回沈晚瓷。他道出了自己其实在学生时期就已经喜欢上了沈晚瓷，这个真相让沈晚瓷震惊不已。她意识到，原来两人之间的情感纠葛早在很久之前就已经开始。\\n最终，两人冰释前嫌，重新走到了一起。这个结局既是对他们情感的圆满交代，也是对他们在困境中坚守和成长的肯定。\\\"\",\"orderWeight\":0,\"backgroundTags\":[],\"tubeSourceType\":3,\"firstOnlineTime\":1758682680929,\"tubeAdoptionStatus\":1,\"tube_biz_type\":\"10\"}",
      "viewCount": 7056,
      "adoptionType": 0,
      "totalEpisodesCountIgnoreStatus": 99
    },
    {
      "collected": false,
      "serialType": 1,
      "firstEpisode": {
        "photoCdnUrl": [
          {
            "cdn": "ali2.a.kwimgs.com",
            "url": "http://ali2.a.kwimgs.com/upic/2025/09/23/15/BMjAyNTA5MjMxNTUyNDhfMTA4NzgzNTI1N18xNzU2MjQyNTg1MjNfMF8z_Bda5e3d083e918e7c407c14b15c340d3b.jpg?tag=1-1758918181-unknown-0-zq0ewootsk-e44c8aadf93db1b3&clientCacheKey=3xmcyxwfkn8fgae.jpg&di=JAmKACWhY6BZDt89hZaTVw==&bp=10001"
          }
        ],
        "linkUrl": "kwai://episode/play?serialId=12982856&serialType=1&photoId=5253167665827256882&selectedPhotoId=5253167665827256882&sourcePhotoPage=attab",
        "name": "第1集",
        "id": 6130651657,
        "photoId": 175624258523,
        "tubeId": 12982856
      },
      "disableViewCountByFilm": false,
      "likeCount": 0,
      "name": "麒麟天医2",
      "id": "5xmhuebjfbtj5pm",
      "finished": true,
      "bizType": 10,
      "totalEpisodesCount": 95,
      "extParams": "{\"display\":{\"poster\":\"\",\"director\":\"\",\"producer\":\"\",\"mainActors\":\"\",\"companyName\":\"\",\"firstPlayDay\":0,\"scriptwriter\":\"\",\"episodeLength\":0,\"copyrightNumber\":\"\",\"institutionName\":\"\",\"producerCompany\":\"\",\"visibleToOutside\":false,\"coProducerCompany\":\"\",\"onlinePlayPlatform\":\"\",\"superviseTheManufacture\":\"\"},\"keyword\":\"麒麟天医2\",\"channelIds\":[],\"screenType\":0,\"contentType\":0,\"description\":\"徐天隐拥有麒麟血的特殊身份，这一身份让他成为了三军统帅萧烈的目标，导致他家破人亡。幸得苏沐晴相救，怎料却害得她身中剧毒。徐天隐历经磨难，终成一代神医，立下赫赫战功，获封麒麟天医，却选择隐居江湖。怎料苏若霜上门退婚，他不得不重出江湖，再掀风云！\",\"orderWeight\":0,\"backgroundTags\":[],\"tubeSourceType\":3,\"firstOnlineTime\":1758614621674,\"tubeAdoptionStatus\":1,\"tube_biz_type\":\"10\"}",
      "viewCount": 30599,
      "adoptionType": 0,
      "totalEpisodesCountIgnoreStatus": 95
    },
    {
      "collected": false,
      "serialType": 1,
      "firstEpisode": {
        "photoCdnUrl": [
          {
            "cdn": "hw2.a.kwimgs.com",
            "url": "http://hw2.a.kwimgs.com/upic/2025/09/23/15/BMjAyNTA5MjMxNTE4NTdfMTA4NzgzNTI1N18xNzU2MjIzNjAwNzVfMF8z_Bd07111cac9ddfbb93dc6256943586559.jpg?tag=1-1758918181-unknown-0-48fhvfilzf-326be3f6ed631c68&clientCacheKey=3xtj6yf9djckszq.jpg&di=JAmKACWhY6BZDt89hZaTVw==&bp=10001"
          }
        ],
        "linkUrl": "kwai://episode/play?serialId=12981820&serialType=1&photoId=5248101116179919976&selectedPhotoId=5248101116179919976&sourcePhotoPage=attab",
        "name": "第1集",
        "id": 6129662710,
        "photoId": 175622360075,
        "tubeId": 12981820
      },
      "disableViewCountByFilm": false,
      "likeCount": 0,
      "name": "萧雨只念晴1",
      "id": "5x7chwchmb7igpc",
      "finished": true,
      "bizType": 10,
      "totalEpisodesCount": 93,
      "extParams": "{\"display\":{\"poster\":\"\",\"director\":\"\",\"producer\":\"\",\"mainActors\":\"\",\"companyName\":\"\",\"firstPlayDay\":0,\"scriptwriter\":\"\",\"episodeLength\":0,\"copyrightNumber\":\"\",\"institutionName\":\"\",\"producerCompany\":\"\",\"visibleToOutside\":false,\"coProducerCompany\":\"\",\"onlinePlayPlatform\":\"\",\"superviseTheManufacture\":\"\"},\"keyword\":\"萧雨只念晴1\",\"channelIds\":[],\"screenType\":0,\"contentType\":0,\"description\":\"故事开始于凌萧宇在一次被追杀中受伤，无意中将传家玉镯交给了顾念晴，并承诺会派人来接她。然而，玉镯意外落入了顾念晴的好友苏冬雪手中，她因此被误认为是凌萧宇的救命恩人，并成为了凌家的女主人。 顾念晴在凌家医院做兼职护士时，偶遇了凌萧宇，两人在一系列误会和冲突中逐渐产生了感情。与此同时，顾念晴的男友江浩炎背叛了她，与另一个女人王丽霞在一起，并且江浩炎还与苏冬雪串通，试图通过苏冬雪嫁入豪门来提升自己的社会地位。 顾念晴的家庭背景也充满了挑战，她的父亲赌博成性，哥哥也是赌徒，家庭负债累累。在一次家庭冲突后，顾念晴被赶出了家门。在苏冬雪和江浩炎的阴谋下，顾念晴在凌氏集团的工作中也遭受了排挤和羞辱。 随着剧情的发展，凌萧宇开始怀疑苏冬雪的真实身份，并逐渐揭露了她和江浩炎的阴谋。\",\"orderWeight\":0,\"backgroundTags\":[],\"tubeSourceType\":3,\"firstOnlineTime\":1758682649238,\"tubeAdoptionStatus\":1,\"tube_biz_type\":\"10\"}",
      "viewCount": 5781,
      "adoptionType": 0,
      "totalEpisodesCountIgnoreStatus": 93
    },
    {
      "collected": false,
      "serialType": 1,
      "firstEpisode": {
        "photoCdnUrl": [
          {
            "cdn": "hw2.a.kwimgs.com",
            "url": "http://hw2.a.kwimgs.com/upic/2025/09/23/15/BMjAyNTA5MjMxNTA0MTZfMTA4NzgzNTI1N18xNzU2MjE1MzMyMTZfMF8z_B9e21e1e7eb2570b70037e5ef82e18d7a.jpg?tag=1-1758918181-unknown-0-t1d4fzjikn-b887d35b8230c5b7&clientCacheKey=3x6aj8nrrphn4bk.jpg&di=JAmKACWhY6BZDt89hZaTVw==&bp=10001"
          }
        ],
        "linkUrl": "kwai://episode/play?serialId=12981330&serialType=1&photoId=5258515689720200578&selectedPhotoId=5258515689720200578&sourcePhotoPage=attab",
        "name": "第1集",
        "id": 6129280666,
        "photoId": 175621533216,
        "tubeId": 12981330
      },
      "disableViewCountByFilm": false,
      "likeCount": 0,
      "name": "善恶昭然",
      "id": "5xqnyddcgh98jj2",
      "finished": true,
      "bizType": 10,
      "totalEpisodesCount": 90,
      "extParams": "{\"display\":{\"poster\":\"\",\"director\":\"\",\"producer\":\"\",\"mainActors\":\"\",\"companyName\":\"\",\"firstPlayDay\":0,\"scriptwriter\":\"\",\"episodeLength\":0,\"copyrightNumber\":\"\",\"institutionName\":\"\",\"producerCompany\":\"\",\"visibleToOutside\":false,\"coProducerCompany\":\"\",\"onlinePlayPlatform\":\"\",\"superviseTheManufacture\":\"\"},\"keyword\":\"善恶昭然\",\"channelIds\":[],\"screenType\":0,\"contentType\":0,\"description\":\"江翎这位英武帅气的男主角，原本是京城江家大少，后因一系列复杂原因被家族逐出，失去双眼并被送入精神病院。在病院中，他遇到了老圣王，获得了神秘力量和特殊技能，同时承担了寻找叛徒、整顿圣门的任务。江翎恢复后，决心清理家族内乱，挫败古武界的阴谋，并在过程中结识了慕蒹葭，一位性格多变的女主角，两人一同为正义而战。\\n剧情围绕江翎的复仇和正义行动展开，涉及多重宴会和冲突场景。江翎在宴会上揭露自己的身份，与反派角色如李天扬、秦沐雨、赵传奇等发生激烈对抗。在这些场合中，江翎展现了他的聪明才智和强大实力，不仅击败了敌人，还赢得了人心。同时，慕蒹葭在剧情中也起到了关键作用，她与江翎并肩作战，共同揭露了家族内的腐败和背叛。\",\"orderWeight\":0,\"backgroundTags\":[],\"tubeSourceType\":3,\"firstOnlineTime\":1758611338146,\"tubeAdoptionStatus\":1,\"tube_biz_type\":\"10\"}",
      "viewCount": 6435,
      "adoptionType": 0,
      "totalEpisodesCountIgnoreStatus": 90
    },
    {
      "collected": false,
      "serialType": 1,
      "firstEpisode": {
        "photoCdnUrl": [
          {
            "cdn": "ali2.a.kwimgs.com",
            "url": "http://ali2.a.kwimgs.com/upic/2025/09/23/14/BMjAyNTA5MjMxNDU1NTNfMTA4NzgzNTI1N18xNzU2MjEwMjc0NTJfMF8z_B1df5edbda052493a97f64233204eee56.jpg?tag=1-1758918181-unknown-0-ox1qxc7i8h-2b81da72539bae01&clientCacheKey=3xmhh5mkfv38eeq.jpg&di=JAmKACWhY6BZDt89hZaTVw==&bp=10001"
          }
        ],
        "linkUrl": "kwai://episode/play?serialId=12980592&serialType=1&photoId=5231494092648779055&selectedPhotoId=5231494092648779055&sourcePhotoPage=attab",
        "name": "第1集",
        "id": 6129024022,
        "photoId": 175621027452,
        "tubeId": 12980592
      },
      "disableViewCountByFilm": false,
      "likeCount": 0,
      "name": "破规之后是新生",
      "id": "5xhhxvhh3f8kgr9",
      "finished": true,
      "bizType": 10,
      "totalEpisodesCount": 60,
      "extParams": "{\"display\":{\"poster\":\"\",\"director\":\"\",\"producer\":\"\",\"mainActors\":\"\",\"companyName\":\"\",\"firstPlayDay\":0,\"scriptwriter\":\"\",\"episodeLength\":0,\"copyrightNumber\":\"\",\"institutionName\":\"\",\"producerCompany\":\"\",\"visibleToOutside\":false,\"coProducerCompany\":\"\",\"onlinePlayPlatform\":\"\",\"superviseTheManufacture\":\"\"},\"keyword\":\"破规之后是新生\",\"channelIds\":[],\"screenType\":0,\"contentType\":0,\"description\":\"新婚夜公公闯房逼背家规，陈雪当场掀被子护身，怼得公公语塞，婆婆跪捧戒尺助虐，丈夫愚孝劝忍，陈雪直接踹飞抱枕怼翻椅子，摔门赶人。后遭家暴夺房、丢工作，她反设局拿财产，离婚开家政发光发热，渣男公婆全落魄！\",\"orderWeight\":0,\"backgroundTags\":[],\"tubeSourceType\":3,\"firstOnlineTime\":1758611341135,\"tubeAdoptionStatus\":1,\"tube_biz_type\":\"10\"}",
      "viewCount": 33516,
      "adoptionType": 0,
      "totalEpisodesCountIgnoreStatus": 60
    },
    {
      "collected": false,
      "serialType": 1,
      "firstEpisode": {
        "photoCdnUrl": [
          {
            "cdn": "ali2.a.kwimgs.com",
            "url": "http://ali2.a.kwimgs.com/upic/2025/09/23/14/BMjAyNTA5MjMxNDA4MTZfMTA4NzgzNTI1N18xNzU2MTgyOTQ1MTFfMF8z_B47fc9cdc422fa443292c05f75591eccc.jpg?tag=1-1758918181-unknown-0-wlsystt6kh-bb3f5b4387c19aa3&clientCacheKey=3xwewarqn693jq9.jpg&di=JAmKACWhY6BZDt89hZaTVw==&bp=10001"
          }
        ],
        "linkUrl": "kwai://episode/play?serialId=12977781&serialType=1&photoId=5216294443835498146&selectedPhotoId=5216294443835498146&sourcePhotoPage=attab",
        "name": "第1集",
        "id": 6127568872,
        "photoId": 175618294511,
        "tubeId": 12977781
      },
      "disableViewCountByFilm": false,
      "likeCount": 0,
      "name": "悠思逐花年",
      "id": "5xr9qvktkiqmbwq",
      "finished": true,
      "bizType": 10,
      "totalEpisodesCount": 65,
      "extParams": "{\"display\":{\"poster\":\"\",\"director\":\"\",\"producer\":\"\",\"mainActors\":\"\",\"companyName\":\"\",\"firstPlayDay\":0,\"scriptwriter\":\"\",\"episodeLength\":0,\"copyrightNumber\":\"\",\"institutionName\":\"\",\"producerCompany\":\"\",\"visibleToOutside\":false,\"coProducerCompany\":\"\",\"onlinePlayPlatform\":\"\",\"superviseTheManufacture\":\"\"},\"keyword\":\"悠思逐花年\",\"channelIds\":[],\"screenType\":0,\"contentType\":0,\"description\":\"大夏战神宁峰在求婚之日，突然接到前线战事告急的消息。因保家卫国，守卫大夏，宁峰情急之下不得放下儿女情长，终止求婚，奔赴战场。七个月后，备受家人欺辱，不愿意成为家族用来联姻工具的谢安然，挺着七个月大的肚子，独自一人离开谢家。\",\"orderWeight\":0,\"backgroundTags\":[],\"tubeSourceType\":3,\"firstOnlineTime\":1758608018839,\"tubeAdoptionStatus\":1,\"tube_biz_type\":\"10\"}",
      "viewCount": 26791,
      "adoptionType": 0,
      "totalEpisodesCountIgnoreStatus": 65
    },
    {
      "collected": false,
      "serialType": 1,
      "firstEpisode": {
        "photoCdnUrl": [
          {
            "cdn": "ali2.a.kwimgs.com",
            "url": "http://ali2.a.kwimgs.com/upic/2025/09/23/14/BMjAyNTA5MjMxNDA1MDBfMTA4NzgzNTI1N18xNzU2MTgxMDY4NjJfMF8z_B49b8d15c63b7682b95409488d0576228.jpg?tag=1-1758918181-unknown-0-0xvalvhdyo-e74de5e16bf97f02&clientCacheKey=3xnschfj2bfqwyc.jpg&di=JAmKACWhY6BZDt89hZaTVw==&bp=10001"
          }
        ],
        "linkUrl": "kwai://episode/play?serialId=12977682&serialType=1&photoId=5250071439616638721&selectedPhotoId=5250071439616638721&sourcePhotoPage=attab",
        "name": "第1集",
        "id": 6127445993,
        "photoId": 175618106862,
        "tubeId": 12977682
      },
      "disableViewCountByFilm": false,
      "likeCount": 0,
      "name": "璃璃原上卿",
      "id": "5xrpfjjvqkf4szk",
      "finished": true,
      "bizType": 10,
      "totalEpisodesCount": 60,
      "extParams": "{\"display\":{\"poster\":\"\",\"director\":\"\",\"producer\":\"\",\"mainActors\":\"\",\"companyName\":\"\",\"firstPlayDay\":0,\"scriptwriter\":\"\",\"episodeLength\":0,\"copyrightNumber\":\"\",\"institutionName\":\"\",\"producerCompany\":\"\",\"visibleToOutside\":false,\"coProducerCompany\":\"\",\"onlinePlayPlatform\":\"\",\"superviseTheManufacture\":\"\"},\"keyword\":\"璃璃原上卿\",\"channelIds\":[],\"screenType\":0,\"contentType\":0,\"description\":\"简介：草原公主江若璃，与部族的三位王子青梅竹马立下婚约，只待成年择一而嫁，共掌草原。可成年礼当日，等来的不是聘礼花轿，而是三人齐齐奔向叶赫兰的残酷现实——她救下的马奴，如今却夺走了她的一切。心灰意冷之际，江若璃毅然选择与神秘的中原“纨绔”楚怀卿和亲。殊不知，楚怀卿的真实身份竟是中原太子，更对她暗恋十年、默默守护。亲手为她披上凤冠霞帔，许她三书六礼、一世独宠。直到出嫁那天，三位王子幡然醒悟。\",\"orderWeight\":0,\"backgroundTags\":[],\"tubeSourceType\":3,\"firstOnlineTime\":1758608460627,\"tubeAdoptionStatus\":1,\"tube_biz_type\":\"10\"}",
      "viewCount": 2480452,
      "adoptionType": 0,
      "totalEpisodesCountIgnoreStatus": 60
    },
    {
      "collected": false,
      "serialType": 1,
      "firstEpisode": {
        "photoCdnUrl": [
          {
            "cdn": "hw2.a.kwimgs.com",
            "url": "http://hw2.a.kwimgs.com/upic/2025/09/22/12/BMjAyNTA5MjIxMjUwNDZfMTA4NzgzNTI1N18xNzU1NDgwMzcxMzNfMF8z_Bb41bd6b7b1268e71527242aaabd1c5a1.jpg?tag=1-1758918181-unknown-0-t3oha5yykt-898fcaa1912676ba&clientCacheKey=3xjepv2j9c4hwkm.jpg&di=JAmKACWhY6BZDt89hZaTVw==&bp=10001"
          }
        ],
        "linkUrl": "kwai://episode/play?serialId=12949125&serialType=1&photoId=5244160464553545116&selectedPhotoId=5244160464553545116&sourcePhotoPage=attab",
        "name": "第1集",
        "id": 6080644934,
        "photoId": 175548037133,
        "tubeId": 12949125
      },
      "disableViewCountByFilm": false,
      "likeCount": 0,
      "name": "云边七载悔归迟",
      "id": "5xnnjecqr79vrze",
      "finished": true,
      "bizType": 10,
      "totalEpisodesCount": 39,
      "extParams": "{\"display\":{\"poster\":\"\",\"director\":\"\",\"producer\":\"\",\"mainActors\":\"\",\"companyName\":\"\",\"firstPlayDay\":0,\"scriptwriter\":\"\",\"episodeLength\":0,\"copyrightNumber\":\"\",\"institutionName\":\"\",\"producerCompany\":\"\",\"visibleToOutside\":false,\"coProducerCompany\":\"\",\"onlinePlayPlatform\":\"\",\"superviseTheManufacture\":\"\"},\"keyword\":\"云边七载悔归迟\",\"channelIds\":[],\"screenType\":0,\"contentType\":0,\"description\":\"剧情简介：豪门少爷季明州为了考验妻子林初桐，故意装穷，六年里，林初桐和女儿安安为生计奔波，艰苦撑起一家，安安一直希望给父亲弹奏摇篮曲，可根本买不起钢琴，结果在琴行发现父亲给青梅的女儿买了一架昂贵钢琴，自此，母女心寒，但还是给了季明州最后一次机会，可在女儿生日宴上，季明州为了陪青梅的女儿依旧没有出现，林初桐母女失望离开出国，季明州追悔莫及。\",\"orderWeight\":0,\"backgroundTags\":[],\"tubeSourceType\":3,\"firstOnlineTime\":1758520622919,\"tubeAdoptionStatus\":1,\"tube_biz_type\":\"10\"}",
      "viewCount": 3264508,
      "adoptionType": 0,
      "totalEpisodesCountIgnoreStatus": 39
    },
    {
      "collected": false,
      "serialType": 1,
      "firstEpisode": {
        "photoCdnUrl": [
          {
            "cdn": "hw2.a.kwimgs.com",
            "url": "http://hw2.a.kwimgs.com/upic/2025/09/21/17/BMjAyNTA5MjExNzIyMDJfMTA4NzgzNTI1N18xNzU0OTM4MzMwNDJfMF8z_B0b98370d51bb1f047cf88c85970e1820.jpg?tag=1-1758918181-unknown-0-2pqzqk0qwt-3b6bbb6c68be0aa6&clientCacheKey=3x9y29d7ggp62uk.jpg&di=JAmKACWhY6BZDt89hZaTVw==&bp=10001"
          }
        ],
        "linkUrl": "kwai://episode/play?serialId=12934981&serialType=1&photoId=5193776446090086292&selectedPhotoId=5193776446090086292&sourcePhotoPage=attab",
        "name": "第1集",
        "id": 6046519789,
        "photoId": 175493833042,
        "tubeId": 12934981
      },
      "disableViewCountByFilm": false,
      "likeCount": 0,
      "name": "夺我天命，我重生嫁太子",
      "id": "5x48kwq7mpcbuia",
      "finished": true,
      "bizType": 10,
      "totalEpisodesCount": 54,
      "extParams": "{\"display\":{\"poster\":\"\",\"director\":\"\",\"producer\":\"\",\"mainActors\":\"\",\"companyName\":\"\",\"firstPlayDay\":0,\"scriptwriter\":\"\",\"episodeLength\":0,\"copyrightNumber\":\"\",\"institutionName\":\"\",\"producerCompany\":\"\",\"visibleToOutside\":false,\"coProducerCompany\":\"\",\"onlinePlayPlatform\":\"\",\"superviseTheManufacture\":\"\"},\"keyword\":\"夺我天命，我重生嫁太子\",\"channelIds\":[],\"screenType\":0,\"contentType\":0,\"description\":\"丞相嫡女时岁安为天命之女，前世助二皇子江玉卿登基却遭其新婚夜杀害，只因他误信庶妹时听雪是天命女。双双重生后，江玉卿求娶时听雪，时岁安坦然放手，大皇子江凛舟顺势求娶。时听雪以莲花花钿冒充胎记，借江玉卿前世记忆伪装预知能力，开启真假天命之争。\",\"orderWeight\":0,\"backgroundTags\":[],\"tubeSourceType\":3,\"firstOnlineTime\":1758522855952,\"tubeAdoptionStatus\":1,\"tube_biz_type\":\"10\"}",
      "viewCount": 8994,
      "adoptionType": 0,
      "totalEpisodesCountIgnoreStatus": 54
    },
    {
      "collected": false,
      "serialType": 1,
      "firstEpisode": {
        "photoCdnUrl": [
          {
            "cdn": "ali2.a.kwimgs.com",
            "url": "http://ali2.a.kwimgs.com/upic/2025/09/20/17/BMjAyNTA5MjAxNzQ2MTBfMTA4NzgzNTI1N18xNzUzOTg5NDM3ODBfMF8z_Bd78f3c0a25d282e79ca10e7029709664.jpg?tag=1-1758918181-unknown-0-dw9pt7fztx-76e0846fe77eadf1&clientCacheKey=3xwn5ar5ib6r7hm.jpg&di=JAmKACWhY6BZDt89hZaTVw==&bp=10001"
          }
        ],
        "linkUrl": "kwai://episode/play?serialId=12912371&serialType=1&photoId=5190398744720869714&selectedPhotoId=5190398744720869714&sourcePhotoPage=attab",
        "name": "第1集",
        "id": 6003283173,
        "photoId": 175398943780,
        "tubeId": 12912371
      },
      "disableViewCountByFilm": false,
      "likeCount": 0,
      "name": "尘缘七秩",
      "id": "5xb38gwux4ey49y",
      "finished": true,
      "bizType": 10,
      "totalEpisodesCount": 60,
      "extParams": "{\"display\":{\"poster\":\"\",\"director\":\"\",\"producer\":\"\",\"mainActors\":\"\",\"companyName\":\"\",\"firstPlayDay\":0,\"scriptwriter\":\"\",\"episodeLength\":0,\"copyrightNumber\":\"\",\"institutionName\":\"\",\"producerCompany\":\"\",\"visibleToOutside\":false,\"coProducerCompany\":\"\",\"onlinePlayPlatform\":\"\",\"superviseTheManufacture\":\"\"},\"keyword\":\"尘缘七秩\",\"channelIds\":[],\"screenType\":0,\"contentType\":0,\"description\":\"\\\"内容梗概：\\n七年前，首富继承人顾廷之遇车祸被陈婉婷所救，相爱生下陈晓晓后，顾廷之昏迷。陈婉婷为救他抽血致瘫，晓晓扛起养家重担。顾家寻孙七年，找到祖孙后开启无底线宠娃模式，却遭顾廷之前女友携子搅局争夺家产。\\\"\",\"orderWeight\":0,\"backgroundTags\":[],\"tubeSourceType\":3,\"firstOnlineTime\":1758427170790,\"tubeAdoptionStatus\":1,\"tube_biz_type\":\"10\"}",
      "viewCount": 46640,
      "adoptionType": 0,
      "totalEpisodesCountIgnoreStatus": 60
    },
    {
      "collected": false,
      "serialType": 1,
      "firstEpisode": {
        "photoCdnUrl": [
          {
            "cdn": "hw2.a.kwimgs.com",
            "url": "http://hw2.a.kwimgs.com/upic/2025/09/20/14/BMjAyNTA5MjAxNDQ0MDdfMTA4NzgzNTI1N18xNzUzODIyMjIxMTVfMF8z_Bdbc404ccb93b07baf6ada3592d2f6744.jpg?tag=1-1758918181-unknown-0-58nalshwnr-ceb937ff70e9af44&clientCacheKey=3x7hadivh4ihkrs.jpg&di=JAmKACWhY6BZDt89hZaTVw==&bp=10001"
          }
        ],
        "linkUrl": "kwai://episode/play?serialId=12906976&serialType=1&photoId=5250071438572044977&selectedPhotoId=5250071438572044977&sourcePhotoPage=attab",
        "name": "第1集",
        "id": 5997711285,
        "photoId": 175382222115,
        "tubeId": 12906976
      },
      "disableViewCountByFilm": false,
      "likeCount": 0,
      "name": "八零换嫁，渔民老公马甲掉了",
      "id": "5xpdti3xgzj77k4",
      "finished": true,
      "bizType": 10,
      "totalEpisodesCount": 57,
      "extParams": "{\"display\":{\"poster\":\"\",\"director\":\"\",\"producer\":\"\",\"mainActors\":\"\",\"companyName\":\"\",\"firstPlayDay\":0,\"scriptwriter\":\"\",\"episodeLength\":0,\"copyrightNumber\":\"\",\"institutionName\":\"\",\"producerCompany\":\"\",\"visibleToOutside\":false,\"coProducerCompany\":\"\",\"onlinePlayPlatform\":\"\",\"superviseTheManufacture\":\"\"},\"keyword\":\"八零换嫁，渔民老公马甲掉了\",\"channelIds\":[],\"screenType\":0,\"contentType\":0,\"description\":\"剧情简介:女主宋时宁上辈子嫁给男配后成了阔太太，妹妹不知道女主的苦楚，在嫉妒之下杀了女主，女主和妹妹双双重生回到了80年代定下婚事的那天。妹妹为了当上阔太太执意嫁给男配，女主替妹妹嫁给了妹妹上辈子逃婚的乡下人男主，女主原来准备过完朴实平淡的一生，没想到男主是隐藏首富，男主收养的两个孩子也乖巧可爱，女主收获了圆满的婚姻，而女配即使被男配冷待，也笃定男配一定会赚大钱，看不起嫁到乡下的女主，后来女配得知了男配上辈子发达都是因为女主的暗中助力，而自己放弃的男主是隐藏首富，后悔莫及想要挽回，男主却表示自己一开始想娶的就是女主，告诉了女主他们都在省城自己就暗恋女主，女主得知男主对自己蓄谋已久，也爱上了男主，过上了幸福生活。\",\"orderWeight\":0,\"backgroundTags\":[],\"tubeSourceType\":3,\"firstOnlineTime\":1758507889805,\"tubeAdoptionStatus\":1,\"tube_biz_type\":\"10\"}",
      "viewCount": 79147,
      "adoptionType": 0,
      "totalEpisodesCountIgnoreStatus": 57
    },
    {
      "collected": false,
      "serialType": 1,
      "firstEpisode": {
        "photoCdnUrl": [
          {
            "cdn": "ali2.a.yximgs.com",
            "url": "http://ali2.a.yximgs.com/upic/2025/09/20/14/BMjAyNTA5MjAxNDQwMjBfMTA4NzgzNTI1N18xNzUzODE5MDMyNzRfMF8z_B2c952b636336f4ccacff5c5b64e78e98.jpg?tag=1-1758918181-unknown-0-wzvb9wnsmu-0e7db89af6e39476&clientCacheKey=3x6u77f32rqmkzc.jpg&di=JAmKACWhY6BZDt89hZaTVw==&bp=10001"
          }
        ],
        "linkUrl": "kwai://episode/play?serialId=12906936&serialType=1&photoId=5221079518069748342&selectedPhotoId=5221079518069748342&sourcePhotoPage=attab",
        "name": "第1集",
        "id": 5997645266,
        "photoId": 175381903274,
        "tubeId": 12906936
      },
      "disableViewCountByFilm": false,
      "likeCount": 0,
      "name": "喜宴断旧情",
      "id": "5x48frcar7zej5q",
      "finished": true,
      "bizType": 10,
      "totalEpisodesCount": 40,
      "extParams": "{\"display\":{\"poster\":\"\",\"director\":\"\",\"producer\":\"\",\"mainActors\":\"\",\"companyName\":\"\",\"firstPlayDay\":0,\"scriptwriter\":\"\",\"episodeLength\":0,\"copyrightNumber\":\"\",\"institutionName\":\"\",\"producerCompany\":\"\",\"visibleToOutside\":false,\"coProducerCompany\":\"\",\"onlinePlayPlatform\":\"\",\"superviseTheManufacture\":\"\"},\"keyword\":\"喜宴断旧情\",\"channelIds\":[],\"screenType\":0,\"contentType\":0,\"description\":\"剧情简介：王俊阳本是全市最优秀的妇产科医生，在一次医院交流活动中，意外撞见自己老婆黄桃体破裂着急做手术，做完手术后王俊阳才知道自己老婆居然跟自己闺蜜老公厮混在一起，心灰意冷的王俊阳决定在老婆闺蜜婚礼上曝光这一切。\",\"orderWeight\":0,\"backgroundTags\":[],\"tubeSourceType\":3,\"firstOnlineTime\":1758353183609,\"tubeAdoptionStatus\":1,\"tube_biz_type\":\"10\"}",
      "viewCount": 4344014,
      "adoptionType": 0,
      "totalEpisodesCountIgnoreStatus": 40
    }
  ]
}


**免责声明**: 本项目仅用于安全研究和学习目的，请勿用于非法用途。使用者需自行承担相关责任。



![6163523eb9b54ba256422d8f2c4ff0d6](https://github.com/user-attachments/assets/24a80507-3b18-45ea-bf9d-1383d30ad247)



