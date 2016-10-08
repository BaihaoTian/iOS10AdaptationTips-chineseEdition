##iOS10AdaptationTips 

### Reference:[iOS10AdaptationTips](https://github.com/ChenYilong/iOS10AdaptationTips)

###Notification

####User Notifications : both a new and old framework

如果你使用如下的命令对比SDK‘iOS10.0’(Xcode8)和SDK‘iOS9.0’的不同，你会发现有6个UIKit类关联通知的类在SDK‘iOS10.0’(Xcode8)废弃了。

```
UIKit9Dir="/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk/System/Library/Frameworks/UIKit.framework"
UIKit10Dir="/Applications/Xcode-beta.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk/System/Library/Frameworks/UIKit.framework"

OptIgnore=--ignore-matching-lines='//.*Copyright'
DIFFBIN=/usr/bin/diff
$DIFFBIN -U 1 -r -x '*.tbd' -x '*.modulemap' $OptIgnore $UIKit9Dir $UIKit10Dir|egrep -C 1 "NS_CLASS_DEPRECATED_IOS.*"|grep interface

```
包含如下这些:

1.UILocalNotification

2.UIMutableUserNotificationAction

3.UIMutableUserNotificationCategory

4.UIUserNotificationAction

5.UIUserNotificationCategory

6.UIUserNotificationSettings

旧的接口也能在SDK‘iOS10.0’(Xcode8)中正常使用，但我们最好开始使用User Notifications framework中的APIs去替代老的接口。

除了这些类以外，这些`handleActionWithIdentifier:forLocalNotification:`,
`handleActionWithIdentifier:forRemoteNotification:`,
`didReceiveLocalNotification:withCompletion :`方法,和WatchKit中的方法
`didReceiveRemoteNotification:withCompletion :`，将会被`handleActionWithIdentifier:forNotification:`,`didReceiveNotification:withCompletion:`代替。此外在WKExtensionDelegate中notification的处理方法，比如`didReceiveRemoteNotification :`和`handleActionWithIdentifier:forRemoteNotification:`。取代使用这些方法的的是，首先穿件一个delegate实现UNUserNotificationCenterDelegate的协议然后选择实现合适的方法。


SDK'iOS 10.0'(Xcode 8) 引入了 从UIKit独立出来的User NOtification framework(UserNotifications.framework),从而支持了传输和处理本地和远程通知。所以可以说他是一个既新鲜又传统的framework。你可以使用这个framework的类基于更具体的情况去有计划的传输本地通知，比如时间或者定位。当通知传送到用户的设备上时，Apps和app extensions 可以使用这个framework去接收和潜在的修改本地通知或远程通知。 

此外，SDK'iOS 10.0'(Xcode 8) 也引入了User Notifications UI framework (UserNotificationsUI.framework) 允许你定制本地或远程通知出现在你的设备上时的外观。你可以通过使用这个framework区定义一个app extension去接受通知的数据然后提供一个与数据符合的可视化外观。当然这个app extension也能响应和这个通知结合的定制的响应动作。

我将分两个部分介绍这个User Notifications framework:

1.本地通知

2.远程通知

####本地通知，可以把任何东西写在一个地方。
一些人可能和这个伙计有同样的问题：


第一个问题几乎是不可能直接解决的，但是通过本地通知从某种角度而言也许是最好的方式去在特定的时间甚至是特定的位置去唤醒你的app。这是因为本地通知就是通过特定的条件比如时间或定位来有计划的传送本地通知。

本地通知有一个局限，你不能触发并执行一段block在通知射出的时候(notification fired 这个老美对弹出通知是这么写的)。 然而你可以通过`UNNotificationAction`添加一个action到你的通知然后利用
`serNotificationCenter(_:didReceive:withCompletionHandler:)`,
`UNUserNotificationCenter.currentNotificationCenter()`
触发并执行一段代码块。也就是说，在不通知用户的情况下，固定的时间在后台执行一段脚本是不可能的。这个特征限制在iOS8以后。                                                                                                                                                                                                                                                                                                                                                          

####通过时间来有计划的发送本地通知
大不同：

1.在SDK‘iOS10.0’（Xcode）中，即使app在前台你也可以展示alert、播放声音、增加角标了。

2.现在当用户点击或者活动通知时，你可以在一个地方处理上述的任何事件了，甚至是这app被杀掉了。

3.支持3DTouch替代手势滑动了。

4.你现在通过仅仅一行代码就能移除特殊的本地通知。

通过OC实现的例子，[iOS10AdaptationTips](https://github.com/ChenYilong/iOS10AdaptationTips)

1.引入 UserNotifications

```objc
///    Notification become independent from UIKit
@import UserNotifications;

```

2.为本地通知请求授权

```objc

UNUserNotificationCenter *center = [UNUserNotificationCenter currentNotificationCenter];
[center requestAuthorizationWithOptions:(UNAuthorizationOptionBadge | UNAuthorizationOptionSound | UNAuthorizationOptionAlert)
                      completionHandler:^(BOOL granted, NSError * _Nullable error) {
                          if (!error) {
                              NSLog(@"request authorization succeeded!");
                              [self showAlert];
                          }
                      }];
                      
```

3.计划本地通知

4.更新应用红点

```objc
    //        //Deliver the notification at 08:30 everyday
    //        NSDateComponents *dateComponents = [[NSDateComponents alloc] init];
    //        dateComponents.hour = 8;
    //        dateComponents.minute = 30;
    //        UNCalendarNotificationTrigger *trigger = [UNCalendarNotificationTrigger triggerWithDateMatchingComponents:dateComponents repeats:YES];

    UNMutableNotificationContent *content = [[UNMutableNotificationContent alloc] init];
    content.title = [NSString localizedUserNotificationStringForKey:@"Elon said:" arguments:nil];
    content.body = [NSString localizedUserNotificationStringForKey:@"Hello Tom！Get up, let's play with Jerry!"
                                                         arguments:nil];
    content.sound = [UNNotificationSound defaultSound];

    /// 4. update application icon badge number
    content.badge = @([[UIApplication sharedApplication] applicationIconBadgeNumber] + 1);
    // Deliver the notification in five seconds.
    UNTimeIntervalNotificationTrigger *trigger = [UNTimeIntervalNotificationTrigger
                                                  triggerWithTimeInterval:5.f repeats:NO];
    UNNotificationRequest *request = [UNNotificationRequest requestWithIdentifier:@"FiveSecond"
                                                                          content:content trigger:trigger];
    /// 3. schedule localNotification
    UNUserNotificationCenter *center = [UNUserNotificationCenter currentNotificationCenter];
    [center addNotificationRequest:request withCompletionHandler:^(NSError * _Nullable error) {
        if (!error) {
            NSLog(@"add NotificationRequest succeeded!");
        }
    }];
```

然后他就会如下图所示：

iOS Version | iOS 10 | iOS 9
-----|------|----
Request Authorization    | ![](http://ocnhrgfjb.bkt.clouddn.com/image/notification/notification-1.jpeg)    | ![](http://ocnhrgfjb.bkt.clouddn.com/image/notification/notification-2.png)
In Background    | ![](http://ocnhrgfjb.bkt.clouddn.com/image/notification/notification-3.jpeg)    | ![](http://ocnhrgfjb.bkt.clouddn.com/image/notification/notification-4.png)
Lock Screen    | ![](http://ocnhrgfjb.bkt.clouddn.com/image/notification/notification-5.jpeg)    | ![](http://ocnhrgfjb.bkt.clouddn.com/image/notification/notification-6.png)
If Repeat by default  |![](http://ocnhrgfjb.bkt.clouddn.com/image/notification/notification-7.jpeg)|![](http://ocnhrgfjb.bkt.clouddn.com/image/notification/notification-8.png)
3D Touch   |![](http://ocnhrgfjb.bkt.clouddn.com/image/notification/notification-9.jpeg)|not support


1.现在通过SDK‘iOS10.0’(Xcode)即使app在前台运行你也能弹出alert，播放声音和增加角标。

2.现在当用户点击或者活动通知时，你可以在一个地方处理上述的任何事件了，即使这app被杀掉了。

3.支持3DTouch替代手势滑动了。

4.你现在通过仅仅一行代码就能移除特殊的本地通知。

``` objc
- (void)stopNotification:(id)sender {
if (SYSTEM_VERSION_GREATER_THAN_OR_EQUAL_TO(@"10.0")) {
    UNUserNotificationCenter *center = [UNUserNotificationCenter currentNotificationCenter];
    // remove all local notification:
//        [center removeAllPendingNotificationRequests];
    // or you can remove specifical local notification:
     [center removePendingNotificationRequestsWithIdentifiers:@[ CYLInviteCategoryIdentifier ]];
} else {
    // remove all local notification:
//        [[UIApplication sharedApplication] cancelAllLocalNotifications];
    // or you can remove specifical local notification:
    NSString *specificalIDToCancel = CYLInviteCategoryIdentifier;
    UILocalNotification *notificationToCancel = nil;
    for(UILocalNotification *aNotif in [[UIApplication sharedApplication] scheduledLocalNotifications]) {
        if([[aNotif.userInfo objectForKey:@"CategoryIdentifier"] isEqualToString:specificalIDToCancel]) {
            notificationToCancel = aNotif;
            break;
        }
    }
    if(notificationToCancel) {
        [[UIApplication sharedApplication] cancelLocalNotification:notificationToCancel];
    }
}
}
```


顺便说下，你可以使用如下代码检查Xcode的版本：

``` objc
#define XCODE_VERSION_GREATER_THAN_OR_EQUAL_TO_8    __has_include(<UserNotifications/UserNotifications.h>)

```

#### 通过定位有计划的发送本地通知

如果你的iphone安装了 Foursquare-app(一款国外app类似大众点评)，当你进入或离开某个地理范围你会收到通知。这是怎么实现的呢？
（？？？？  没下文了 翻译的我有点方）




###一些小问题

> Check dependencies Signing for Your-Prject-Name requires a development team. Select a development team in the Target Editor. Warning: The Copy Bundle Resources build phase contains this target's Info.plist file '/Users//Info.plist'. Code signing is required for product type 'Application' in SDK 'iOS 10.0'

####ATS 
Reference:[Security and Privacy Enhancements](https://developer.apple.com/library/prerelease/content/releasenotes/General/WhatsNewIniOS/Articles/iOS10.html#//apple_ref/doc/uid/TP40017084-SW3)

译者自己补充（iOS 9中默认非HTTPS的网络是被禁止的，当然我们也可以把NSAllowsArbitraryLoads设置为YES禁用ATS。不过iOS 10从2017年1月1日起苹果不允许我们通过这个方法跳过ATS，也就是说强制我们用HTTPS，如果不这样的话提交App可能会被拒绝。但是我们可以通过NSExceptionDomains来针对特定的域名开放HTTP可以容易通过审核。）

#### Security 安全

#####Access privacy-sensitive data 隐私及敏感数据访问权限

在你访问照相机、通讯录、等等隐私以及敏感数据之前，你必须请求授权。否则你的app会在你尝试访问这些隐私时崩溃。Xcode会log这些：
>This app has crashed because it attempted to access privacy-sensitive data without a usage description. The app's Info.plist must contain an NSContactsUsageDescription key with a string value explaining to the user how the app uses this data.

怎么处理这个问题呢？

打开你工程中名叫`info.plist`的文件，右键点击选择`opening as Source Code`，把下面的代码粘贴进去。或者你可以使用默认的`Property List`打开`info.plist`，点击add按钮，当你输入`Privacy - `Xcode会给你自动补全的建议，用上下键去选择吧。

```objc
 <!-- 🖼 Photo Library -->
    <key>NSPhotoLibraryUsageDescription</key>
    <string></string>

    <!-- 📷 Camera -->
    <key>NSCameraUsageDescription</key>
    <string></string>

    <!-- 🎤 Microphone -->
    <key>NSMicrophoneUsageDescription</key>
    <string></string>

    <!-- 📍 Location -->
    <key>NSLocationUsageDescription</key>
    <string></string>

    <!-- 📍 Location When In Use -->
    <key>NSLocationWhenInUseUsageDescription</key>
    <string></string>

    <!-- 📍 Location Always -->
    <key>NSLocationAlwaysUsageDescription</key>
    <string></string>

    <!-- 📆 Calendars -->
    <key>NSCalendarsUsageDescription</key>
    <string></string>

    <!-- ⏰ Reminders -->
    <key>NSRemindersUsageDescription</key>
    <string></string>

    <!-- 🏊 Motion -->
    <key>NSMotionUsageDescription</key>
    <string></string>

    <!-- 💊 Health Update -->
    <key>NSHealthUpdateUsageDescription</key>
    <string></string>

    <!-- 💊 Health Share -->
    <key>NSHealthShareUsageDescription</key>
    <string></string>

    <!-- ᛒ🔵 Bluetooth Peripheral -->
    <key>NSBluetoothPeripheralUsageDescription</key>
    <string></string>

    <!-- 🎵 Media Library -->
    <key>NSAppleMusicUsageDescription</key>
    <string></string>
```


如果这样做没起作用，试着去请求后台模式的授权。

```objc
<key>UIBackgroundModes</key>
<array>
    <!-- something you should use in background -->
    <string>location</string>
</array>
```

或者去 `target -> Capabilities -> Background Modes -> open the background Modes`

![](http://ocnhrgfjb.bkt.clouddn.com/image/notification/pic-1.png)
然后clean你的工程，run起来。


#### 你可以在Xib或Storyboard同时使用AutoresizingMask和Autolayout Constraints布局
虽然iOS6已经给出兼容性让开发者同时使用AutoresizingMask和Autolayout Constraints，Xcode会把AutoresizingMask代码转换成Constraints，但是针对Xib或Storyboard却不兼容。

```objc
@interface UIView (UIConstraintBasedCompatibility) 

/* By default, the autoresizing mask on a view gives rise to constraints that fully determine 
 the view's position. This allows the auto layout system to track the frames of views whose 
 layout is controlled manually (through -setFrame:, for example).
 When you elect to position the view using auto layout by adding your own constraints, 
 you must set this property to NO. IB will do this for you.
 */
@property(nonatomic) BOOL translatesAutoresizingMaskIntoConstraints NS_AVAILABLE_IOS(6_0); // Default YES
@end
```
但是现在Xcode可以帮你把AutoresizingMask代码转换成Constraints，所以你可以在Xib或Storyboard中同时使用AutoresizingMask和Autolayout Constraints。

#### iOS 版本检查

在你的app中不要再使用如下方式检查iOS系统版本

```objc
#define IsIOS7 ([[[[UIDevice currentDevice] systemVersion] substringToIndex:1] intValue]>=7)

```
这会始终返回NO，`substringToIndex:1`在SDK‘iOS 10.0’(Xcode)中等于SDK‘iOS 1.0’

使用如下的代码替换吧

>Objective-C

```objc
#define SYSTEM_VERSION_EQUAL_TO(v)                  ([[[UIDevice currentDevice] systemVersion] compare:v options:NSNumericSearch] == NSOrderedSame)
#define SYSTEM_VERSION_GREATER_THAN(v)              ([[[UIDevice currentDevice] systemVersion] compare:v options:NSNumericSearch] == NSOrderedDescending)
#define SYSTEM_VERSION_GREATER_THAN_OR_EQUAL_TO(v)  ([[[UIDevice currentDevice] systemVersion] compare:v options:NSNumericSearch] != NSOrderedAscending)
#define SYSTEM_VERSION_LESS_THAN(v)                 ([[[UIDevice currentDevice] systemVersion] compare:v options:NSNumericSearch] == NSOrderedAscending)
#define SYSTEM_VERSION_LESS_THAN_OR_EQUAL_TO(v)     ([[[UIDevice currentDevice] systemVersion] compare:v options:NSNumericSearch] != NSOrderedDescending)

```

OR

```objc
//App Deployment Target should be beyond 8.0:
if ([[NSProcessInfo processInfo] isOperatingSystemAtLeastVersion:(NSOperatingSystemVersion){.majorVersion = 9, .minorVersion = 1, .patchVersion = 0}]) {
    NSLog(@"Hello from > iOS 9.1");
}

// Using short-form for the struct, we can make things somewhat more compact:
if ([NSProcessInfo.processInfo isOperatingSystemAtLeastVersion:(NSOperatingSystemVersion){9,3,0}]) {
    NSLog(@"Hello from > iOS 9.3");
}
```

OR

```objc
//App Deployment Target should be beyond 2.0:
if (NSFoundationVersionNumber > NSFoundationVersionNumber_iOS_9_0) {
    // do stuff for iOS 9 and newer
} else {
    // do stuff for older versions than iOS 9
}
```

SDK 'iOS 10.0' (Xcode 8) 给出了更多的版本号码甚至是特征版本(小版本)

```objc
#define NSFoundationVersionNumber_iOS_9_0 1240.1
#define NSFoundationVersionNumber_iOS_9_1 1241.14
#define NSFoundationVersionNumber_iOS_9_2 1242.12
#define NSFoundationVersionNumber_iOS_9_3 1242.12
#define NSFoundationVersionNumber_iOS_9_4 1280.25
#define NSFoundationVersionNumber_iOS_9_x_Max 1299
```

>Swift

```swift
if NSProcessInfo().isOperatingSystemAtLeastVersion(NSOperatingSystemVersion(majorVersion: 10, minorVersion: 0, patchVersion: 0)) {
    // modern code
}
```

OR:

```swift
if #available(iOS 10.0, *) {
    // modern code
} else {
    // Fallback on earlier versions
}
```

Reference:[Efficient iOS Version Checking](https://pspdfkit.com/blog/2016/efficient-iOS-version-checking/)

