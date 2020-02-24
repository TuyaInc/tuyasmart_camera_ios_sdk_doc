## 设备配网

涂鸦 IPC 硬件 Wi-Fi 主要支持以下配网模式：

* 快连模式（Smart Config 模式）
* 热点模式（AP模式）
* 二维码模式

> 二维码模式较为简单，建议使用二维码配网，如果设备扫描不了二维码，再尝试快连模式。

由于云云对接和 SDK 对接，集成的配网 SDK 不一样，设备激活的流程和代码会有少许不同。

**快连模式**和**热点模式**这两种配网方式和涂鸦其他设备的配网方式一样，云云对接方案接入的是裁剪版的配网 SDK，可以参考[云云对接配网文档](https://github.com/TuyaInc/tuyasmart_ios_activator_sdk/blob/master/README-zh.md)。

涂鸦智能 SDK 对接方案，可以参考文档[设备配网](https://tuyainc.github.io/tuyasmart_home_ios_sdk_doc/zh-hans/resource/Activator.html#%E8%AE%BE%E5%A4%87%E9%85%8D%E7%BD%91)。

下面着重介绍一下智能摄像机特有的二维码配网模式。

### 二维码配网

**二维码模式配网流程：**

```sequence

Title: 二维码配网

participant APP
participant SDK
participant Device
participant Service

Note over APP: 连上路由器
Note over Device: Wifi灯快闪

APP->SDK: 获取token
SDK->Service: 获取token
Service-->SDK: 返回token
SDK-->APP: 返回token

APP-->APP: 将 ssid/pwd/token 拼接字符串生成二维码
Device-->APP: 摄像机扫描二维码获取 WI-FI 信息和 Token
Device->Service: 去激活设备

APP->SDK: 开始配网 ssid/pwd/token

Service-->Device: 激活成功
Device-->SDK: 激活成功
SDK-->APP: 激活成功

```

#### 获取token

开始配网之前，SDK需要在联网状态下从涂鸦云获取配网 Token，然后将 Wi-Fi 的 ssid 与密码一起生成二维码。Token的有效期为10分钟，且配置成功后就会失效（再次配网需要重新获取）。设备配网需要依赖于家庭，设备必须绑定在某个家庭下，所有获取配网 Token 时，需要传入一个家庭的 ID 作为参数，设备使用这个 Token 激活成功后，就会绑定在这个家庭的设备列表中。

__Objective-C__

```objc
- (void)getToken {
    [[TuyaSmartActivator sharedInstance] getTokenWithHomeId:homeId success:^(NSString *token) {
        NSLog(@"getToken success: %@", token);
        // TODO: startConfigWiFi
    } failure:^(NSError *error) {
        NSLog(@"getToken failure: %@", error.localizedDescription);
    }];
}
```

__Swift__

```swift
func getToken() {
    TuyaSmartActivator.sharedInstance()?.getTokenWithHomeId(homeId, success: { (token) in
        print("getToken success: \(token)")
        // TODO: startConfigWiFi
    }, failure: { (error) in
        if let e = error {
            print("getToken failure: \(e)")
        }
    })
}
```

#### 生成二维码

获取到配网 Token 后，还需要有期望设备连接的 Wi-Fi 的 ssid 和密码，通过下面的方式拼接成字符串，然后根据这个字符串生成一个二维码图片。

__Objective-C__

```objc
NSDictionary *dictionary = @{
@"s": self.ssid,
@"p": self.pwd,
@"t": self.token
};
NSData *jsonData = [NSJSONSerialization dataWithJSONObject:dictionary options:0 error:nil];
self.wifiJsonStr = [[NSString alloc] initWithData:jsonData encoding:NSUTF8StringEncoding];
```

__Swift__

```swift
let dictionary = [
    "s": self.ssid,
    "p": self.pwd,
    "t": self.token
]
let jsonData = JSONSerialization.data(withJSONObject: dictionary, options: JSONSerialization.WritingOptions.Element)
self.wifiJsonStr = String(data: jsonData, encoding: String.Encoding.utf8)
```

#### 开始配网

使用上面生成的 wifiJsonStr 生成二维码，确定设备处于配网状态，将二维码对准摄像头，设备捕捉到二维码信息后会发出提示音。此时通过以下接口开始监听配网结果。

__Objective-C__

```objc
[[TuyaSmartActivator sharedInstance] startConfigWiFi:TYActivatorModeQRCode ssid:self.ssid password:self.pwd token:self.token timeout:100];
```

__Swift__

```swift
TuyaSmartActivator.sharedInstance()?.startConfigWiFi(TYActivatorModeQRCode, ssid: self.ssid, password: self.pwd, token: self.token, timeout: 100)
```

#### 停止配网

配网过程中，可使用下面的方法停止配网。

__Objective-C__

```objc
[[TuyaSmartActivator sharedInstance] stopConfigWiFi];
```

__Swift__

```swift
TuyaSmartActivator.sharedInstance()?.stopConfigWiFi()
```

#### 设备激活回调

配网结果通过代理 ```TuyaSmartActivatorDelegate```回调，代理方法如下：

__Objective-C__

```objc
// deviceModel 则为配网成功的设备的信息
- (void)activator:(TuyaSmartActivator *)activator didReceiveDevice:(TuyaSmartDeviceModel *)deviceModel error:(NSError *)error;
```

__Swift__

```swift
func activator(_ activator: TuyaSmartActivator!, didReceiveDevice deviceModel: TuyaSmartDeviceModel!, error: Error!)
```

#### 云云对接方案 Token

云云对接方案集成的 mini 配网 SDK，没有直接获取配网 Token 的接口，需要接入方从自己的云端获取配网 Token。接入方的云端会从涂鸦云端获取配网 Token，具体可以参考[云云对接文档-配网管理](https://docs.tuya.com/zh/iot/open-api/api-list/api/paring-management)。

### 绑定模式

涂鸦智能设备支持强、中、弱三种绑定模式，设备成功激活到对应账号的家庭中后，不同的绑定模式，解除绑定需要不同的校验方式。

* **强绑定**：需要前一个用户在 App 中移除设备，才可以重新配网绑定到另一个账号中；
* **中绑定**：无需前一个用户在 App 中移除设备，即可重新配网绑定到另一个账号中，但是会向前一个账号的家庭组/默认组管理员发送PUSH通知消息；
* **弱绑定**：无需前一个用户在 App 中移除设备，即可重新配网绑定到另一个账号中。

由于 IPC 有音视频传输，涉及到的隐私比较多，默认是强绑定模式，且不可以切换成其他模式，如果接入方有强烈需求，且评估过修改成其他模式的影响，可向涂鸦开发者平台提交工单处理。
