---
title: 'iOS获取设备唯一ID'
layout: post
categories: 技术
tags:
    - iOS
---

在开发过程中，我们常常需要获取设备的一个唯一标识，以便后续做相应的处理或者数据分析。在iOS平台中，随着各个版本的变迁，这一简单需求也充满了与苹果的斗智斗勇。

### UDID ###

UDID的全称是Unique Device Identifier，它是苹果iOS设备的唯一识别码，由40个字符的字母和数字组成。在很多需要限制一台设备一个账号的应用中经常会用到。在iOS5中可以获取到设备的UDID，iOS5之后该方法标记为废弃，在iOS7中已经移除了该方法。这个是目前为止唯一可以确认唯一的标示符。苹果禁止这个API是出去安全的考虑，许多开发者把UDID跟用户的真实姓名、密码、住址、其它数据关联起来。从一个应用可能拿不到这个用户的隐私数据，但是从多个应用，就有可能拼凑出用户的隐私数据。

获取方法

```objectivec
    [[UIDevice currentDevice] uniqueIdentifier]
```

### 网卡地址 ###

MAC地址在网络上用来区分设备的唯一性，接入网络的设备都有一个MAC地址，他们肯定都是不同的，是唯一的。一部iPhone上可能有多个MAC地址，包括WIFI的、SIM的等，但是iTouch和iPad上就有一个WIFI的，因此只需获取WIFI的MAC地址就好了，也就是en0的地址。MAC地址就如同我们身份证上的身份证号码，具有全球唯一性。但在iOS7之后，如果请求Mac地址都会返回一个固定值。

### IDFV ###

iOS 6.0系统新增用于替换uniqueIdentifier的接口。是给Vendor标识用户用的，每个设备在所属同一个Vender的应用里，都有相同的值。其中的Vender是指应用提供商，但准确点说，是通过BundleID的DNS反转的前两部分进行匹配，如果相同就是同一个Vender，例如对于com.somecompany.appone,com.somecompany.apptwo这两个BundleID来说，就属于同一个Vender，共享同一个idfv的值。和idfa不同的是，idfv的值是一定能取到的，所以非常适合于作为内部用户行为分析的主id，来标识用户，替代OpenUDID。如果用户将属于此Vender的所有App卸载，则idfv的值会被重置，即再重装此Vender的App，idfv的值和之前不同。

获取方法

```objectivec
    [[[UIDevice currentDevice] identifierForVendor] UUIDString]
```

### IDFA ###

广告标示符，适用于对外：例如广告推广，换量等跨应用的用户追踪等。但如果用户完全重置系统（(设置程序 -> 通用 -> 还原 -> 还原位置与隐私) ，这个广告标示符会重新生成。另外如果用户明确的还原广告(设置程序-> 通用 -> 关于本机 -> 广告 -> 还原广告标示符) ，那么广告标示符也会重新生成。注意：如果程序在后台运行，此时用户“还原广告标示符”，然后再回到程序中，此时获取广 告标示符并不会立即获得还原后的标示符。必须要终止程序，然后再重新启动程序，才能获得还原后的广告标示符。在同一个设备上的所有App都会取到相同的值，是苹果专门给各广告提供商用来追踪用户而设的，用户可以在 设置 -> 隐私 -> 广告追踪 里重置此id的值，或限制此id的使用。

获取方法

```objectivec
    [[[ASIdentifierManager sharedManager] advertisingIdentifier] UUIDString];
```

### KeyChain ###

iOS整个系统有一个KeyChain，每个程序都可以往KeyChain中记录数据，而且只能读取到自己程序记录在KeyChain中的数据。而且就算我们程序删除掉，系统经过升级以后再安装回来，依旧可以获取到之前存入的数据。因此我们可以将设备的唯一id的字符串存储到KeyChain中，然后下次直接从KeyChain获取UUID字符串。注意，KeyChain中的数据并不是一直都在，在更改了应用的provisioning profile后，KeyChain中的数据是会丢失的。虽然这种情况很少，但还是有这样的可能性。

### 综合方案 ###

使用IDFA或者UUID的方法，产生一个唯一标识字符串。这个字符串有可能发生变化，因此需要将它存入KeyChain中，以后都从KeyChain中查询

首先，我们获取IDFA作为设备的唯一标识，这里会先检查AdSupport框架是否可用。

```objectivec
    - (NSString *)appleIFA {
        NSString *ifa = nil;
        Class ASIdentifierManagerClass = NSClassFromString(@"ASIdentifierManager");
        if (ASIdentifierManagerClass) { // a dynamic way of checking if AdSupport.framework is available
            SEL sharedManagerSelector = NSSelectorFromString(@"sharedManager");
            id sharedManager = ((id (*)(id, SEL))[ASIdentifierManagerClass methodForSelector:sharedManagerSelector])(ASIdentifierManagerClass, sharedManagerSelector);
            
            SEL advertisingIdentifierSelector = NSSelectorFromString(@"advertisingIdentifier");
            NSUUID *advertisingIdentifier = ((NSUUID* (*)(id, SEL))[sharedManager methodForSelector:advertisingIdentifierSelector])(sharedManager, advertisingIdentifierSelector);
            
            ifa = [advertisingIdentifier UUIDString];
        }
        return ifa;
    }
```

如果IDFA不可用，那么产生一个UUID作为设备唯一标识：

```objectivec
    - (NSString *)randomUUID {
        return [[NSUUID UUID] UUIDString];
    }
```

这样，我们就有了一个可以唯一标识设备的字符串。接下来，将生成的唯一标识存入KeyChain，这里封装一下关于KeyChain的读写方法：

```objectivec
    #pragma mark - Keychain

    - (NSMutableDictionary *)keychainItemForKey:(NSString *)key service:(NSString *)service {
        NSMutableDictionary *keychainItem = [[NSMutableDictionary alloc] init];
        keychainItem[(__bridge id)kSecClass] = (__bridge id)kSecClassGenericPassword;
        keychainItem[(__bridge id)kSecAttrAccessible] = (__bridge id)kSecAttrAccessibleAlways;
        keychainItem[(__bridge id)kSecAttrAccount] = key;
        keychainItem[(__bridge id)kSecAttrService] = service;
        return keychainItem;
    }

    - (OSStatus)setValue:(NSString *)value forKeychainKey:(NSString *)key inService:(NSString *)service {
        NSMutableDictionary *keychainItem = [self keychainItemForKey:key service:service];
        keychainItem[(__bridge id)kSecValueData] = [value dataUsingEncoding:NSUTF8StringEncoding];
        return SecItemAdd((__bridge CFDictionaryRef)keychainItem, NULL);
    }

    - (NSString *)valueForKeychainKey:(NSString *)key service:(NSString *)service {
        OSStatus status;
        NSMutableDictionary *keychainItem = [self keychainItemForKey:key service:service];
        keychainItem[(__bridge id)kSecReturnData] = (__bridge id)kCFBooleanTrue;
        keychainItem[(__bridge id)kSecReturnAttributes] = (__bridge id)kCFBooleanTrue;
        CFDictionaryRef result = nil;
        status = SecItemCopyMatching((__bridge CFDictionaryRef)keychainItem, (CFTypeRef *)&result);
        if (status != noErr) {
            return nil;
        }
        NSDictionary *resultDict = (__bridge_transfer NSDictionary *)result;
        NSData *data = resultDict[(__bridge id)kSecValueData];
        if (!data) {
            return nil;
        }
        return [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
    }

    - (void)deleteForKeychainKey:(NSString *)key inService:(NSString *)service {
        NSMutableDictionary *keychainItem = [self keychainItemForKey:key service:service];
        SecItemDelete((CFDictionaryRef)keychainItem);
    }
```

由于KeyChain中的数据在更改了应用的provisioning profile后会丢失，这里在NSUserDefaults里再存储一份，作为备份。同样，提供NSUserDefaults的简便封装方法：

```objectivec
    #pragma mark - NSUserDefaults

    - (BOOL)setValue:(NSString *)value forUserDefaultsKey:(NSString *)key {
        [[NSUserDefaults standardUserDefaults] setObject:value forKey:key];
        return [[NSUserDefaults standardUserDefaults] synchronize];
    }

    - (NSString *)valueForUserDefaultsKey:(NSString *)key {
        return [[NSUserDefaults standardUserDefaults] objectForKey:key];
    }
```

那么，就可以调用deviceId方法，获取设备的唯一标识了。

```objectivec
    #pragma mark - DeviceId

    - (NSString *)deviceId {
        if (!_deviceId) {
            _deviceId = [self valueForKeychainKey:DEVICE_ID_KEY service:DEVICE_ID_KEY];
        }
        if (!_deviceId) {
            _deviceId = [self valueForUserDefaultsKey:DEVICE_ID_KEY];
        }
        if (!_deviceId) {
            _deviceId = [self appleIFA];
        }
        if (!_deviceId) {
            _deviceId = [self randomUUID];
        }
        [self save:_deviceId];
        return _deviceId;
    }

    - (void)save:(NSString *)deviceId {
        if (![self valueForUserDefaultsKey:DEVICE_ID_KEY]) {
            [self setValue:deviceId forUserDefaultsKey:DEVICE_ID_KEY];
        }
        if (![self valueForKeychainKey:DEVICE_ID_KEY service:DEVICE_ID_KEY]) {
            [self setValue:deviceId forKeychainKey:DEVICE_ID_KEY inService:DEVICE_ID_KEY];
        }
    }
```

