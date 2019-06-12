---
title: 'iOS崩溃日志收集'
layout: post
categories: 技术
tags:
    - iOS
---

iOS应用在使用中不可避免的会有闪退现象。经常性的闪退会导致一部分用户选择卸载APP，这样会带来很大损失。当应用崩溃时，如何收集崩溃信息怎么进行分析就成为一个很大的问题。之前的项目中，我们使用了友盟进行崩溃信息的收集。不过，为了能够第一时间知晓APP崩溃的情况，我们希望APP能够在崩溃后将崩溃日志发送到服务端，服务端在接收到崩溃日志后报警，让开发能够尽早介入。

## 使用系统自带的Crash信息 ##

iOS崩溃的引起主要有两方面原因，一是未捕获的异常，二是底层的错误信号，这方面常见的是非法的内存访问，比如访问了一个已经释放了的内存地址。这两类都可以通过注册一个处理函数来集中处理。在处理函数中，可以拿到堆栈信息，以及其他的一些信息，将这些信息收集并发送到服务端，方便后续的分析和排查。

```objectivec
    static int s_fatal_signals[] = {
        SIGABRT,
        SIGILL,
        SIGSEGV,
        SIGFPE,
        SIGBUS,
        SIGPIPE
    };

    static int s_fatal_signal_num = sizeof(s_fatal_signals) / sizeof(s_fatal_signals[0]);

    static NSString * const UncaughtExceptionHandlerSignalKey = @"UncaughtExceptionHandlerSignalKey";

    void SignalHandler(int signal) {
        NSString *sysName = [UIDevice currentDevice].systemName; // iOS
        NSString *sysVersion = [UIDevice currentDevice].systemVersion; // 系统版本号
        NSString *model = [UIDevice currentDevice].model; // iPhone / iPod touch / iPad
        
        NSDictionary *infoDictionary = [[NSBundle mainBundle] infoDictionary];
        NSString *appName = [infoDictionary objectForKey:@"CFBundleName"]; // app名称
        NSString *appVersion = [infoDictionary objectForKey:@"CFBundleShortVersionString"]; // app版本
        NSString *appBuild = [infoDictionary objectForKey:@"CFBundleVersion"]; // app build版本
        
        // exception stask
        NSMutableString *mstr = [[NSMutableString alloc] init];
        [mstr appendString:@"(\n"];
        void* callstack[128];
        int i, frames = backtrace(callstack, 128);
        char** strs = backtrace_symbols(callstack, frames);
        for (i = 0; i <frames; ++i) {
            [mstr appendFormat:@"%s\n", strs[i]];
        }
        [mstr appendString:@")"];
        
        NSString *info = [NSString stringWithFormat:@"SystemName === %@\nSystemVersion === %@\nDeviceModel === %@\nAppName === %@\nAppVersion === %@\nAppBuild === %@\nExceptionStack === %@\nSignal === %d\n", sysName, sysVersion, model, appName, appVersion, appBuild, mstr, signal];
        
        [[MBOAnalytics sharedInstance] logCrash:info];
    }

    void ExceptionHandler(NSException *exception) {
        NSString *sysName = [UIDevice currentDevice].systemName; // iOS
        NSString *sysVersion = [UIDevice currentDevice].systemVersion; // 系统版本号
        NSString *model = [UIDevice currentDevice].model; // iPhone / iPod touch / iPad
        
        NSDictionary *infoDictionary = [[NSBundle mainBundle] infoDictionary];
        NSString *appName = [infoDictionary objectForKey:@"CFBundleName"]; // app名称
        NSString *appVersion = [infoDictionary objectForKey:@"CFBundleShortVersionString"]; // app版本
        NSString *appBuild = [infoDictionary objectForKey:@"CFBundleVersion"]; // app build版本
        
        NSString *exceptionName = [exception name]; // 异常名称
        NSString *exceptionReason = [exception reason]; // 出现异常的原因
        NSArray *exceptionStack = [exception callStackSymbols]; // 异常堆栈信息

        NSString *exceptionInfo = [NSString stringWithFormat:@"SystemName === %@\nSystemVersion === %@\nDeviceModel === %@\nAppName === %@\nAppVersion === %@\nAppBuild === %@\nExceptionName === %@\nExceptionReason === %@\nExceptionStack === %@\n", sysName, sysVersion, model, appName, appVersion, appBuild, exceptionName, exceptionReason, exceptionStack];
        
        [[MBOAnalytics sharedInstance] logCrash:exceptionInfo];
    }

    @implementation MBOAppDelegate (Crash)

    - (void)initCrashReport {
        // linux错误信号捕获
        for (int i = 0; i < s_fatal_signal_num; ++i) {
            signal(s_fatal_signals[i], SignalHandler);
        }
        
        // objective-c未捕获异常的捕获
        NSSetUncaughtExceptionHandler(&ExceptionHandler);
    }

    @end
```

## 使用PLCrashReporter收集 ##

PLCrashReporter是一个开源的崩溃日志收集的库，很多收集崩溃日志的第三方服务都是基于PLCrashReporter实现的。

PLCrashReporter可以通过Pod引入：

```objectivec
    pod ‘PLCrashReporter’, ‘~> 1.2’
```

或者直接拖入到Xcode工程中。

使用时，引入头文件：

```objectivec
    #import <CrashReporter/CrashReporter.h>
    #import <CrashReporter/PLCrashReportTextFormatter.h>
```

在didFinishLaunchingWithOptions方法中进行初始化：

```objectivec
    - (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
        
        [self initCrashReport];
        
        ......
        
        return YES;
    }

    - (void)initCrashReport {
        
        PLCrashReporter *crashReporter = [PLCrashReporter sharedReporter];
        NSError *err;
        // Check if we previously crashed
        if ([crashReporter hasPendingCrashReport]) {
            [self handleCrashReport];
        }
        // Enable the Crash Reporter
        if (![crashReporter enableCrashReporterAndReturnError: &err]) {
            NSLog(@"Warning: Could not enable crash reporter: %@", err);
        }
    }
```

如果有收集到崩溃日志，我们将它发送到服务端：

```objectivec
    - (void)handleCrashReport {
        PLCrashReporter *crashReporter = [PLCrashReporter sharedReporter];
        NSData *crashData;
        NSError *error;
        
        // Try loading the crash report
        crashData = [crashReporter loadPendingCrashReportDataAndReturnError:&error];
        if (crashData == nil) {
            NSLog(@"Could not load crash report: %@", error);
            [crashReporter purgePendingCrashReport];
            return;
        }
        
        // We could send the report from here, but we'll just print out some debugging info instead
        PLCrashReport *report = [[PLCrashReport alloc] initWithData:crashData error:&error];
        if (report == nil) {
            NSLog(@"Could not parse crash report");
            [crashReporter purgePendingCrashReport];
            return;
        }
        
        // send the report
        NSString *time = [NSString stringWithFormat:@"Crashed on %@", report.systemInfo.timestamp];
        NSString *info = [NSString stringWithFormat:@"Crashed with signal %@ (code %@, address=0x%" PRIx64 ")", report.signalInfo.name, report.signalInfo.code, report.signalInfo.address];
        NSString *humanReadText = [PLCrashReportTextFormatter stringValueForCrashReport:report withTextFormat:PLCrashReportTextFormatiOS];
        
        intptr_t slide = calculateSlideAddress();
        NSString *slideAddress = [NSString stringWithFormat:@"Slide Address: 0x%lx", slide];
        
        NSString *content = [NSString stringWithFormat:@"%@\n%@\n%@\n%@", time, info, humanReadText, slideAddress];
        NSLog(@"%@", content);
        
        [MBOBaseService serviceWithUrl:MBO_URL_UploadAppErrorLog resultClass:nil params:@{ @"log": content } success:^(MBOBaseService *baseService, id result) {
            
            [crashReporter purgePendingCrashReport];
            
        } failure:^(MBOBaseService *baseService, NSError *error) {
            NSLog(@"ERROR, failed to upload crash log");
        }];
        
    }
```

崩溃日志中的堆栈信息全是16进制的内存地址，根本无法定位到出错的语句。这就需要一个符号化的过程。

符号化需要用到dSYM文件。Xcode编译项目后，会看到一个同名的dSYM文件，dSYM是保存16进制函数地址映射信息的中转文件，我们调试的symbols都会包含在这个文件中，并且每次编译项目的时候都会生成一个新的dSYM文件，位于/Users/<用户名>/Library/Developer/Xcode/Archives目录下，对于每一个发布版本我们都很有必要保存对应的Archives文件.

每一个xxx.app和xxx.app.dSYM文件都有对应的UUID，crash文件(指的是通过工具转换过的文件)也有自己的UUID，只要这三个文件的UUID一致，我们就可以通过他们解析出正确的错误函数信息了。

查看xxx.app 文件的 UUID，在terminal中输入命令 ：

```objectivec
    dwarfdump --uuid xxx.app/xxx (xxx代表你的项目名)
```

查看xxx.app.dSYM文件的UUID，在terminal中输入命令：

```objectivec
    dwarfdump --uuid xxx.app.dSYM (xxx代表你的项目名)
```

crash文件内第一行Incident Identifier就是该crash文件的UUID。

Xcode 7.3中symbolicatecrash工具存放的路径是：

```objectivec
    /Applications/Xcode.app/Contents/SharedFrameworks/DVTFoundation.framework/Versions/A/Resources/symbolicatecrash
```

建议拷贝出来放到一个专门的文件夹下。

设置DEVELOPER_DIR:

```objectivec
    export DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer
```

然后执行:

```objectivec
    ./symbolicatecrash crash.log xxx.app.dSYM > result.log (xxx代表你的项目名)
```

result.log中就是符号化后的日志文件了。