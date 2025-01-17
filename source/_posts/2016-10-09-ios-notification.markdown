---
title: 'iOS 10 推送适配'
layout: post
categories: 技术
tags:
    - iOS
---

iOS 10中对推送相关的API进行了很大调整，统一了以前杂乱的API，现在可以使用独立的UserNotifications.framework来集中管理和使用iOS系统中通知的功能。之前绝大部分通知相关的API都已经被标记为deprecated。

实现通知的第一步是导入UserNotifications头文件，并实现UNUserNotificationCenterDelegate协议。在AppDelegate中，

```objectivec

    #import <UserNotifications/UserNotifications.h>
```

第二步在 (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions 方法中添加 APNs 注册的代码：

```objectivec
    if ([[UIDevice currentDevice].systemVersion floatValue] >= 10.0) {
#if __IPHONE_OS_VERSION_MAX_ALLOWED >= __IPHONE_10_0 // Xcode 8编译会调用
        UNUserNotificationCenter *center = [UNUserNotificationCenter currentNotificationCenter];
        center.delegate = self.userNotificationCenterDelegate;
        [center requestAuthorizationWithOptions:(UNAuthorizationOptionBadge | UNAuthorizationOptionSound | UNAuthorizationOptionAlert | UNAuthorizationOptionCarPlay) completionHandler:^(BOOL granted, NSError *_Nullable error) {
            if (!error) {
                NSLog(@"UNUserNotificationCenter request authorization succeeded!");
            }
        }];
        
        [[UIApplication sharedApplication] registerForRemoteNotifications];
#else // Xcode 7编译会调用
        UIUserNotificationType types = (UIUserNotificationTypeAlert | UIUserNotificationTypeSound | UIUserNotificationTypeBadge);
        UIUserNotificationSettings *settings = [UIUserNotificationSettings settingsForTypes:types categories:nil];
        [[UIApplication sharedApplication] registerForRemoteNotifications];
        [[UIApplication sharedApplication] registerUserNotificationSettings:settings];
#endif
    } else if ([[[UIDevice currentDevice] systemVersion] floatValue] >= 8.0) {
        UIUserNotificationType types = (UIUserNotificationTypeAlert | UIUserNotificationTypeSound | UIUserNotificationTypeBadge);
        UIUserNotificationSettings *settings = [UIUserNotificationSettings settingsForTypes:types categories:nil];
        [[UIApplication sharedApplication] registerForRemoteNotifications];
        [[UIApplication sharedApplication] registerUserNotificationSettings:settings];
    } else {
        UIRemoteNotificationType apn_type = (UIRemoteNotificationType)(UIRemoteNotificationTypeAlert | UIRemoteNotificationTypeSound | UIRemoteNotificationTypeBadge);
        [[UIApplication sharedApplication] registerForRemoteNotificationTypes:apn_type];
    }
```

获取deviceToken的方法和注册失败时的回调方法都没有变化，

```objectivec

    - (void)application:(UIApplication *)application
    didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken {
        NSLog(@"%@", [NSString stringWithFormat:@"Device Token: %@", deviceToken]);
    }

    - (void)application:(UIApplication *)application
    didFailToRegisterForRemoteNotificationsWithError:(NSError *)error {
        NSLog(@"did Fail To Register For Remote Notifications With Error: %@", error);
    }
```

在旧版本中，通知的接收方法是：

```objectivec
    - (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo {
        NSLog(@"iOS6及以下系统，收到通知:%@", [self logDic:userInfo]);
    }

    - (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo
    fetchCompletionHandler:(void (^)(UIBackgroundFetchResult))completionHandler {

        NSLog(@"iOS7及以上系统，收到通知:%@", [self logDic:userInfo]);
        completionHandler(UIBackgroundFetchResultNewData);
    }
```

而在iOS 10中接收通知的方法有所不同，UNUserNotificationCenterDelegate中有两个新增的方法用来处理通知的接收和点击事件。此外，苹果把本地通知和远程通知合二为一，用来区分本地通知和远程通知的类型是UNPushNotificationTrigger。

接收通知的方法如下：

```objectivec
    // iOS 10收到通知
    - (void)userNotificationCenter:(UNUserNotificationCenter *)center willPresentNotification:(UNNotification *)notification withCompletionHandler:(void (^)(UNNotificationPresentationOptions options))completionHandler{
        NSDictionary * userInfo = notification.request.content.userInfo;
        UNNotificationRequest *request = notification.request; // 收到推送的请求
        UNNotificationContent *content = request.content; // 收到推送的消息内容
        NSNumber *badge = content.badge;  // 推送消息的角标
        NSString *body = content.body;    // 推送消息体
        UNNotificationSound *sound = content.sound;  // 推送消息的声音
        NSString *subtitle = content.subtitle;  // 推送消息的副标题
        NSString *title = content.title;  // 推送消息的标题

        if([notification.request.trigger isKindOfClass:[UNPushNotificationTrigger class]]) {
            NSLog(@"iOS10 前台收到远程通知:%@", [self logDic:userInfo]);
        } else {
            // 判断为本地通知
            NSLog(@"iOS10 前台收到本地通知:{\\\\nbody:%@，\\\\ntitle:%@,\\\\nsubtitle:%@,\\\\nbadge：%@，\\\\nsound：%@，\\\\nuserInfo：%@\\\\n}",body,title,subtitle,badge,sound,userInfo);
        }
        completionHandler(UNNotificationPresentationOptionBadge|UNNotificationPresentationOptionSound|UNNotificationPresentationOptionAlert); // 需要执行这个方法，选择是否提醒用户，有Badge、Sound、Alert三种类型可以设置
    }
```

处理通知点击的方法如下：

```objectivec
    // 通知的点击事件
    - (void)userNotificationCenter:(UNUserNotificationCenter *)center didReceiveNotificationResponse:(UNNotificationResponse *)response withCompletionHandler:(void(^)())completionHandler{

        NSDictionary * userInfo = response.notification.request.content.userInfo;
        UNNotificationRequest *request = response.notification.request; // 收到推送的请求
        UNNotificationContent *content = request.content; // 收到推送的消息内容
        NSNumber *badge = content.badge;  // 推送消息的角标
        NSString *body = content.body;    // 推送消息体
        UNNotificationSound *sound = content.sound;  // 推送消息的声音
        NSString *subtitle = content.subtitle;  // 推送消息的副标题
        NSString *title = content.title;  // 推送消息的标题
        if([response.notification.request.trigger isKindOfClass:[UNPushNotificationTrigger class]]) {
            NSLog(@"iOS10 收到远程通知:%@", [self logDic:userInfo]);
        } else {
            // 判断为本地通知
            NSLog(@"iOS10 收到本地通知:{\\\\nbody:%@，\\\\ntitle:%@,\\\\nsubtitle:%@,\\\\nbadge：%@，\\\\nsound：%@，\\\\nuserInfo：%@\\\\n}",body,title,subtitle,badge,sound,userInfo);
        }

        // Warning: UNUserNotificationCenter delegate received call to -userNotificationCenter:didReceiveNotificationResponse:withCompletionHandler: but the completion handler was never called.
        completionHandler();  // 系统要求执行这个方法
    }
```