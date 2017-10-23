---
layout: post
title: AFNetworking 3.1.0 源码学习（一）AFNetworkReachabilityManager
date: 2017-10-23 09:18:24.000000000 +09:00
tags: iOS

---

前面[这篇文章](http://ethanx.me/2017/10/afnetworking-study-overview/)已经对AFNetworking框架有个宏观层面上大概了解，接下来将详细地学习框架的每个模块。

本文基于[AFNetworking 3.1.0](https://github.com/AFNetworking/AFNetworking)进行学习，我们这次先学习`AFNetworkReachabilityManager`，这是一个用于监听网络通信状态的模块，读者可以打开源代码跟着文章思路来学习。

## 头文件部分

先来看看`AFNetworkReachabilityManager.h`头文件

```objc
#if !TARGET_OS_WATCH
#import <SystemConfiguration/SystemConfiguration.h>
```

检查系统类型，如果是除`watchOS`系统外的其他系统（`iOS`、`macOS`、`tvOS`），就导入系统框架`SystemConfiguration/SystemConfiguration.h`。这个系统框架提供`网络通信状态相关`的API（这里与`#if`配对的`#endif`在文件末尾，说明这个模块对`watchOS`来说是不可用的）。

```objc
typedef NS_ENUM(NSInteger, AFNetworkReachabilityStatus) {
    AFNetworkReachabilityStatusUnknown          = -1,  //未知网络
    AFNetworkReachabilityStatusNotReachable     = 0,  //网络不可达
    AFNetworkReachabilityStatusReachableViaWWAN = 1,  //通过蜂窝网络连接，如4G移动网络
    AFNetworkReachabilityStatusReachableViaWiFi = 2,  //通过WiFi网络连接
};
```

这里定义了一个`NS_ENUM`，列出了四种网络状态类型。`当某个类或对象具有不同状态时，应采用NS_ENUM来定义`

```objc
NS_ASSUME_NONNULL_BEGIN
NS_ASSUME_NONNULL_END  
```

在这两句代码之间声明的属性、方法的参数和方法的返回值，都设定为`nonnull`。

接下来就到公开的属性和方法部分，先看属性。

```objc
//记录当前网络状态
@property (readonly, nonatomic, assign) AFNetworkReachabilityStatus networkReachabilityStatus;	
//记录网络是否可达
@property (readonly, nonatomic, assign, getter = isReachable) BOOL reachable;
//记录是否通过蜂窝网络连接
@property (readonly, nonatomic, assign, getter = isReachableViaWWAN) BOOL reachableViaWWAN;
//记录是否通过WiFi连接网络
@property (readonly, nonatomic, assign, getter = isReachableViaWiFi) BOOL reachableViaWiFi;
```

这里声明了四个只读属性，外界调用者通过它们的getter来获取。`公开的只读BOOL类型属性，比较规范的就是通过getter=isXXXX来设定getter方法名称，提高代码的可读性`。

```objc
//返回默认的单例manager
+ (instancetype)sharedManager;
//创建新的manager，这种类方法一般叫Convenience Initializers(便利性构造方法)
+ (instancetype)manager;
//通过域名创建新的manager，同样是Convenience Initializers
+ (instancetype)managerForDomain:(NSString *)domain;
//通过IP地址创建新的manager，同样是Convenience Initializers
+ (instancetype)managerForAddress:(const void *)address;
- (instancetype)initWithReachability:(SCNetworkReachabilityRef)reachability NS_DESIGNATED_INITIALIZER;
- (nullable instancetype)init NS_UNAVAILABLE;
```

这里提供了六个初始化方法，用来获取或创建新的`AFNetworkReachabilityManager`。前面四个都是类方法，后面两个是实例方法。

其中要注意的是`initWithReachability:`这个方法， 这里出现`NS_DESIGNATED_INITIALIZER`这个宏定义，它指明这是一个`Designated Initializer（指定构造方法）`，其他所有初始化方法都必须要调用这个方法，否则编译器会提示错误；另外这个初始化方法还接收一个`SCNetworkReachabilityRef`对象作为参数，这个对象保存创建某个网络连接返回的引用，`AFNetworkReachabilityManager`通过这个引用来监听这个网络连接状态。

另外一个初始化实例方法`init`，这里用`NS_UNAVAILABLE`指明外界调用者不能显式调用init来初始化。

```objc
- (void)startMonitoring;
- (void)stopMonitoring;
```

开始监听和停止监听的方法

```objc
- (NSString *)localizedNetworkReachabilityStatusString;
```

返回描述网络连接状态的国际化字符串

```objc
- (void)setReachabilityStatusChangeBlock:(nullable void (^)(AFNetworkReachabilityStatus status))block;
```

设置网络连接状态变化时的回调`block`，这个`block`没有返回值，但有一个`AFNetworkReachabilityStatus`参数，保存网络变化后的连接状态类型。外界调用者传递`block`时，通过判断`status`，来决定不同网络连接状态进行不同的操作。`这是监听网络变化的回调的方式之一`

```objc
//网络连接状态变化时会发送通知
FOUNDATION_EXPORT NSString * const AFNetworkingReachabilityDidChangeNotification;
FOUNDATION_EXPORT NSString * const AFNetworkingReachabilityNotificationStatusItem;
```

这里声明了一个通知标识符，外界调用者通过监听这个通知，当网络发生变化时，外界调用者会收到通知，并且通知的useInfo会携带一个NSDictionary类型对象，其中key为`AFNetworkingReachabilityNotificationStatusItem`，value则是`AFNetworkReachabilityStatus`枚举类型的网络连接状态。外界调用者接收到这个通知后，判断这个value的值来进行不同的操作。`这是另一种监听网络变化的回调的方式`。

另外这里还出现了`FOUNDATION_EXPORT`，这个宏定义与`#define`很类似，都是用于定义常量。前者在检测字符串是否相等时效率更快，并且可以使用`==`来进行比较，它比较的是指针地址，而`#define`只能使用`isEqualToString:`方法来比较字符串，它比较字符串的每个字符是否相等。

```objc
FOUNDATION_EXPORT NSString * AFStringFromNetworkReachabilityStatus(AFNetworkReachabilityStatus status);
```

最后声明了一个C语言函数，这个函数接收一个`AFNetworkReachabilityStatus`类型参数，并返回一个描述网络连接状态的国际化字符串。实际上，上面的`localizedNetworkReachabilityStatusString`方法就是调用这里的`AFStringFromNetworkReachabilityStatus`函数。

到此，`AFNetworkReachabilityManager.h`头文件分析学习结束，接下来就是详细学习`AFNetworkReachabilityManager.m`实现文件部分。

## 实现文件部分

首先看看.m文件包含了哪些头文件

```objc
#import "AFNetworkReachabilityManager.h"
#if !TARGET_OS_WATCH

#import <netinet/in.h>
#import <netinet6/in6.h>
#import <arpa/inet.h>
#import <ifaddrs.h>
#import <netdb.h>
```

第一个不用说了，接着是判断系统类型，同样是对watchOS不可用，而后面包含的文件都是为了往后的`socket编程`做准备。至于`socket通信`的基础知识和编程实践，可以看看这篇文章：[Linux Socket编程（不限Linux）](http://www.cnblogs.com/skynet/archive/2010/12/12/1903949.html)

```objc
//定义网络连接状态变化的通知
NSString * const AFNetworkingReachabilityDidChangeNotification = @"com.alamofire.networking.reachability.change";
//接收到通知后，用于获取网络状态的key
NSString * const AFNetworkingReachabilityNotificationStatusItem = @"AFNetworkingReachabilityNotificationStatusItem";
//网络状态变化的回调block类型
typedef void (^AFNetworkReachabilityStatusBlock)(AFNetworkReachabilityStatus status);
```

这几个没什么说的，看注释

```objc
NSString * AFStringFromNetworkReachabilityStatus(AFNetworkReachabilityStatus status) {
    switch (status) {
        case AFNetworkReachabilityStatusNotReachable:
            return NSLocalizedStringFromTable(@"Not Reachable", @"AFNetworking", nil);
        case AFNetworkReachabilityStatusReachableViaWWAN:
            return NSLocalizedStringFromTable(@"Reachable via WWAN", @"AFNetworking", nil);
        case AFNetworkReachabilityStatusReachableViaWiFi:
            return NSLocalizedStringFromTable(@"Reachable via WiFi", @"AFNetworking", nil);
        case AFNetworkReachabilityStatusUnknown:
        default:
            return NSLocalizedStringFromTable(@"Unknown", @"AFNetworking", nil);
    }
}
```

对应.h文件的最后一个方法的实现，根据不同的status返回描述网络状态的国际化字符串。注意这里一个比较规范的习惯就是`每个定义常量字符串的地方都进行国际化处理`。

```objc
static AFNetworkReachabilityStatus AFNetworkReachabilityStatusForFlags(SCNetworkReachabilityFlags flags) {
  	//网络是否可达
    BOOL isReachable = ((flags & kSCNetworkReachabilityFlagsReachable) != 0);
  	//是否需要建立连接
    BOOL needsConnection = ((flags & kSCNetworkReachabilityFlagsConnectionRequired) != 0);
  	//是否可以自动连接
    BOOL canConnectionAutomatically = (((flags & kSCNetworkReachabilityFlagsConnectionOnDemand ) != 0) || ((flags & kSCNetworkReachabilityFlagsConnectionOnTraffic) != 0));
  	//在不需要用户手动设置的情况下，是否可以连接
    BOOL canConnectWithoutUserInteraction = (canConnectionAutomatically && (flags & kSCNetworkReachabilityFlagsInterventionRequired) == 0);
  	//在网络可达的前提下，如果不需要建立连接或者不需要用户手动设置，就表示网络可以连接
    BOOL isNetworkReachable = (isReachable && (!needsConnection || canConnectWithoutUserInteraction));
	//根据前面的BOOL值返回status
    AFNetworkReachabilityStatus status = AFNetworkReachabilityStatusUnknown;
    if (isNetworkReachable == NO) {
        status = AFNetworkReachabilityStatusNotReachable;
    }
#if	TARGET_OS_IPHONE
    else if ((flags & kSCNetworkReachabilityFlagsIsWWAN) != 0) {
        status = AFNetworkReachabilityStatusReachableViaWWAN;
    }
#endif
    else {
        status = AFNetworkReachabilityStatusReachableViaWiFi;
    }

    return status;
}
```

这是一个私有的C函数。根据`SCNetworkReachabilityFlags`参数返回网络连接状态status。 很多开源框架的类私有方法都是用的C函数，这里我觉得主要是考虑效率问题，尤其是有可能被多次调用的方法，用C函数实现能提高APP的响应速度。

```objc
static void AFPostReachabilityStatusChange(SCNetworkReachabilityFlags flags, AFNetworkReachabilityStatusBlock block) {
  	//根据flags获取status
    AFNetworkReachabilityStatus status = AFNetworkReachabilityStatusForFlags(flags);
  	//调用回调block和发送网络状态变化通知
    dispatch_async(dispatch_get_main_queue(), ^{
        if (block) {
            block(status);
        }
        NSNotificationCenter *notificationCenter = [NSNotificationCenter defaultCenter];
        NSDictionary *userInfo = @{ AFNetworkingReachabilityNotificationStatusItem: @(status) };
        [notificationCenter postNotificationName:AFNetworkingReachabilityDidChangeNotification object:nil userInfo:userInfo];
    });
}
```

这里涉及到`GCD多线程编程`，作者通过`异步`的方式在`主队列`添加任务，因为`主队列`是`串行队列`，任务是一个接着一个地执行，保证`回调block`和`通知`的发送与接受是按照正确的顺序进行。注意，这里调用的`block(status)`是参数的`block`，而这个函数是私有函数，所以这个`block`是内部传递过来的，并不是外界调用者传递过来的，如果觉得迷糊可以先放着，继续往下看。

```objc
static void AFNetworkReachabilityCallback(SCNetworkReachabilityRef __unused target, SCNetworkReachabilityFlags flags, void *info) {
    AFPostReachabilityStatusChange(flags, (__bridge AFNetworkReachabilityStatusBlock)info);
}


static const void * AFNetworkReachabilityRetainCallback(const void *info) {
  	//持有回调block
    return Block_copy(info);
}

static void AFNetworkReachabilityReleaseCallback(const void *info) {
    if (info) {
      	//释放回调block
        Block_release(info);
    }
}
```

这里的`AFNetworkReachabilityCallback`函数调用上面的`AFPostReachabilityStatusChange`函数，只是把一些参数传递过去。在`AFNetworkReachabilityCallback`这个函数的第一个参数中，使用了`__unused`，这个关键字告诉编译器，如果该参数没有使用就不需要提示警告。另外还有`__bridge`，这个关键字的作用是将`C指针`转换为`Objective-C指针`，即转换为`OC对象`，这样在ARC有效的情况下，编译器可以帮助我们处理引用计数的问题。另外两个是用于持有和释放block的私有函数。这里的三个私有函数，加上上边的`AFPostReachabilityStatusChange`函数，都是这个类关键函数，现在看可能觉得迷糊，先放着，继续往下看。

```objc
@interface AFNetworkReachabilityManager ()
//保存某个网络连接的引用  
@property (readonly, nonatomic, assign) SCNetworkReachabilityRef networkReachability;
//网络连接状态
@property (readwrite, nonatomic, assign) AFNetworkReachabilityStatus networkReachabilityStatus;
//外界传递过来的回调block
@property (readwrite, nonatomic, copy) AFNetworkReachabilityStatusBlock networkReachabilityStatusBlock;
@end
```

类扩展定义了三个属性，`networkReachability`为只读属性，`networkReachabilityStatus`属性设置了`readwrite`，对应.h文件的`readonly`，保证该属性对外不可修改。

接下来就是六个初始化方法的详细实现，我们一个一个来。

```objc
- (instancetype)init NS_UNAVAILABLE
{
    return nil;
}
```

这个初始化方法用到`NS_UNAVAILABLE`，说明这个初始化方法是不可用的，直接放回nil。

```objc
- (instancetype)initWithReachability:(SCNetworkReachabilityRef)reachability {
    self = [super init];
    if (!self) {
        return nil;
    }
	//设置实例变量，CFRetain函数是增加reachability引用计数
    _networkReachability = CFRetain(reachability);
  	//初始化网络状态的值
    self.networkReachabilityStatus = AFNetworkReachabilityStatusUnknown;

    return self;
}
```

这个是`Designated Initializer（指定构造方法）`，其他所有初始化方法最终都要调用这个方法。

```objc
+ (instancetype)managerForDomain:(NSString *)domain {
  	//通过域名创建一个SCNetworkReachabilityRef变量
    SCNetworkReachabilityRef reachability = SCNetworkReachabilityCreateWithName(kCFAllocatorDefault, [domain UTF8String]);
	//调用Designated Initializer创建manager
    AFNetworkReachabilityManager *manager = [[self alloc] initWithReachability:reachability];
    //释放reachability
    CFRelease(reachability);

    return manager;
}
```

这是一个通过域名创建`AFNetworkReachabilityManager`的初始化方法，注意这里的`CFRelease`方法是与`initWithReachability:`初始化方法的`CFRetain`方法对应，使得创建的`reachability`的引用计数为1，即`manager`持有`reachability`。

```objc
+ (instancetype)managerForAddress:(const void *)address {
  	//通过IP地址创建一个SCNetworkReachabilityRef变量
    SCNetworkReachabilityRef reachability = SCNetworkReachabilityCreateWithAddress(kCFAllocatorDefault, (const struct sockaddr *)address);
  	//调用Designated Initializer创建manager
    AFNetworkReachabilityManager *manager = [[self alloc] initWithReachability:reachability];
	//释放reachability
    CFRelease(reachability);
    
    return manager;
}
```

跟刚才的初始化方法类似，只不过这里是通过IP地址来创建`manager`。这里出现一个`sockaddr`类型，它是一个结构体，用于保存socket通信过程中IP地址相关的信息（IP地址长度，地址家族，地址内容等）。综合上边两个方法，我们发现`SCNetworkReachabilityRef` 有两个初始化方法：

* SCNetworkReachabilityCreateWithName
* SCNetworkReachabilityCreateWithAddress

```objc
+ (instancetype)manager
{
#if (defined(__IPHONE_OS_VERSION_MIN_REQUIRED) && __IPHONE_OS_VERSION_MIN_REQUIRED >= 90000) || (defined(__MAC_OS_X_VERSION_MIN_REQUIRED) && __MAC_OS_X_VERSION_MIN_REQUIRED >= 101100)
    struct sockaddr_in6 address;
    bzero(&address, sizeof(address));
    address.sin6_len = sizeof(address);
    address.sin6_family = AF_INET6;
#else
    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_len = sizeof(address);
    address.sin_family = AF_INET;
#endif
    return [self managerForAddress:&address];
}
```

这个方法先判断当前的iOS系统是否大于9.0或者macOS系统是否大于10.11，如果是则创建`IPv6`的地址，否则创建`IPv4`的地址。	这里涉及到比较底层的socket网络编程。最后调用`managerForAddress:`方法创建`manager`。另外，`#if...#else...#endif`这种条件编译指令，会在编译的时候做出判断，只对符合条件的代码做编译。

```objc
+ (instancetype)sharedManager {
    static AFNetworkReachabilityManager *_sharedManager = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _sharedManager = [self manager];
    });

    return _sharedManager;
}
```

最后一个初始化方法是返回一个`单例manager`。这个方法中用到了`GCD`，`dispatch_once`的`block`只会调用一次，第一次调用`sharedManager`方法时，会通过`[self manager]`创建`manager`，往后再次调用`sharedManager`则只会返回第一次创建的结果。

至此，六个初始化方法就讲完了。接下来再看看其他代码。

```objc
- (void)dealloc {
  	//停止监听
    [self stopMonitoring];
    
    if (_networkReachability != NULL) {
      	//释放manager持有的SCNetworkReachabilityRef变量
        CFRelease(_networkReachability);
    }
}
```

当前`manager`被释放回收时做的一些处理工作。

```objc
- (BOOL)isReachable {
    return [self isReachableViaWWAN] || [self isReachableViaWiFi];
}

- (BOOL)isReachableViaWWAN {
    return self.networkReachabilityStatus == AFNetworkReachabilityStatusReachableViaWWAN;
}

- (BOOL)isReachableViaWiFi {
    return self.networkReachabilityStatus == AFNetworkReachabilityStatusReachableViaWiFi;
}
```

对应.h文件暴露的三个BOOL只读属性的getter方法。

接下来看整个类的核心监听代码

```objc
- (void)startMonitoring {
  	//先停止当前进行的监听
    [self stopMonitoring];
	//如果networkReachability属性为空，即没有连接引用，则返回
    if (!self.networkReachability) {
        return;
    }
	//创建一个对self的弱引用，并且类型为self的类型，即AFNetworkReachabilityManager
  	// __typeof返回对应的类型
    __weak __typeof(self)weakSelf = self;
  	//创建一个回调block，这个block是SCNetworkReachabilityRef网络状态变化时被直接调用的
    AFNetworkReachabilityStatusBlock callback = ^(AFNetworkReachabilityStatus status) {
      	//创建一个强引用，保证weakSelf在block调用过程中不被释放
        __strong __typeof(weakSelf)strongSelf = weakSelf;
		//设置网络状态变化后的状态
        strongSelf.networkReachabilityStatus = status;
      	//调用外界调用者传递过来的回调block，传递status到外界调用者
        if (strongSelf.networkReachabilityStatusBlock) {
            strongSelf.networkReachabilityStatusBlock(status);
        }

    };
	//创建SCNetworkReachabilityContext
    SCNetworkReachabilityContext context = {0, (__bridge void *)callback, AFNetworkReachabilityRetainCallback, AFNetworkReachabilityReleaseCallback, NULL};
  	//设置SCNetworkReachabilityRef网络状态变化时的回调
    SCNetworkReachabilitySetCallback(self.networkReachability, AFNetworkReachabilityCallback, &context);
  	//SCNetworkReachabilityRef对象只有加入到RunLoop才能执行监听，并且以kCFRunLoopCommonModes模式进行
    SCNetworkReachabilityScheduleWithRunLoop(self.networkReachability, CFRunLoopGetMain(), kCFRunLoopCommonModes);
	//异步派发，任务将会在后台执行
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0),^{
        SCNetworkReachabilityFlags flags;
      	//立即获取当前的网络状态
        if (SCNetworkReachabilityGetFlags(self.networkReachability, &flags)) {
          	//调用AFPostReachabilityStatusChange方法传递flags
            AFPostReachabilityStatusChange(flags, callback);
        }
    });
}
```

看到这里估计会一头雾水，这里稍微有点复杂，有很多callback回调，我们慢慢来理清。前面几句代码比较简单，我们看下这里创建的`callback`局部变量`block`，这个`block`是当`SCNetworkReachabilityRef`底层检测到网络变化时的回调，用于告诉`AFNetworkReachabilityManager`网络发生变化。而这个`block`里调用的`strongSelf.networkReachabilityStatusBlock`则是外界调用者传递过来的`block`，即告诉外界网络发生变化了。接着往下，创建了一个`SCNetworkReachabilityContext`，我们看下`SCNetworkReachabilityContext`的定义：

```objc
typedef struct {
  	//版本号
	CFIndex		version;	
  	//回调函数指针info
	void *		__nullable info;
  	//用于对info做retain操作的函数指针
	const void	* __nonnull (* __nullable retain)(const void *info);
  	//用于对info做release操作的函数指针
	void		(* __nullable release)(const void *info);
  	//描述info
	CFStringRef	__nonnull (* __nullable copyDescription)(const void *info);
} SCNetworkReachabilityContext;
```

我们再看回`AFNetworkReachabilityManager.m`的代码

```objc
SCNetworkReachabilityContext context = {0, (__bridge void *)callback, AFNetworkReachabilityRetainCallback, AFNetworkReachabilityReleaseCallback, NULL};
```

将之前创建的`callback`以及`AFNetworkReachabilityRetainCallback`和`AFNetworkReachabilityReleaseCallback` 作为参数来创建`context`。`callback`就是回调（不是外界调用者传递过来的），而`AFNetworkReachabilityRetainCallback`和`AFNetworkReachabilityReleaseCallback`则是对这个`callback`进行`retain`和`release`操作的函数（这时可以看回前面的这两个私有函数，应该就懂了）。继续往下，我们先看看`SCNetworkReachabilitySetCallback`函数的定义：

```objc
Boolean
SCNetworkReachabilitySetCallback		(
  						//SCNetworkReachabilityRef变量，指向网络连接的引用
						SCNetworkReachabilityRef			target,
  						//一个回调
						SCNetworkReachabilityCallBack	__nullable	callout,
  						//SCNetworkReachabilityContext上下文
						SCNetworkReachabilityContext	* __nullable	context
						)				__OSX_AVAILABLE_STARTING(__MAC_10_3,__IPHONE_2_0);
```

再看回`AFNetworkReachabilityManager.m`的代码

```objc
SCNetworkReachabilitySetCallback(self.networkReachability, AFNetworkReachabilityCallback, &context);
```

第一个参数和最后一个参数分别是`SCNetworkReachabilityRef`指向网络连接的引用和刚刚创建的`SCNetworkReachabilityContext`上下文，比较疑惑的是为什么这里会出现一个回调`AFNetworkReachabilityCallback`呢？我们先看看`SCNetworkReachabilitySetCallback`方法第二个参数的类型定义：

```objc
typedef void (*SCNetworkReachabilityCallBack)	(
						SCNetworkReachabilityRef			target,
						SCNetworkReachabilityFlags			flags,
						void			     *	__nullable	info
						);
```

可以看到这个回调应该有三个参数，第一个是`SCNetworkReachabilityRef`变量，第二个是`SCNetworkReachabilityFlags`变量，还记得前面的`AFNetworkReachabilityStatusForFlags`方法吗？就是根据这个`flags`来判断网络连接状态的。第三个参数是一个函数指针`info`，这个`info`与前面的`SCNetworkReachabilityContext`的`info`是一样的，也就是创建`context`时的第二个参数`callback`。而`AFNetworkReachabilityCallback`的作用是啥？我的理解就是为了适应`SCNetworkReachabilityCallBack`这个类型，因为`AFNetworkReachabilityCallback`函数里只是把两个参数传递给`AFPostReachabilityStatusChange`。注意`AFNetworkReachabilityCallback`的`info`参数和`AFPostReachabilityStatusChange`的`block`参数都是前面`startMonitoring`方法里创建的`callback`（这时可以看回前面的两个私有函数`AFNetworkReachabilityCallback`和`AFPostReachabilityStatusChange`）。这部分比较绕，需要多看看多想想才能理清。

我们接着看回`startMonitoring`方法：

```objc
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0),^{
        SCNetworkReachabilityFlags flags;
        if (SCNetworkReachabilityGetFlags(self.networkReachability, &flags)) {
            AFPostReachabilityStatusChange(flags, callback);
        }
    });
```

因为前面的设置都是针对网络状态变化时候触发，这里是立即获取当前网络状态。

```objc
- (void)stopMonitoring {
    if (!self.networkReachability) {
        return;
    }
	//从RunLoop中删除self.networkReachability
    SCNetworkReachabilityUnscheduleFromRunLoop(self.networkReachability, CFRunLoopGetMain(), kCFRunLoopCommonModes);
}
```

停止监听的方法，没什么说的。

```objc
#pragma mark -

- (NSString *)localizedNetworkReachabilityStatusString {
    return AFStringFromNetworkReachabilityStatus(self.networkReachabilityStatus);
}

#pragma mark -

- (void)setReachabilityStatusChangeBlock:(void (^)(AFNetworkReachabilityStatus status))block {
    self.networkReachabilityStatusBlock = block;
}
```

这两个方法对应.h文件的公开方法，一个是返回描述网络状态的国际化字符串，另外一个就是保存外界调用者传递过来的`block`。

```objc
+ (NSSet *)keyPathsForValuesAffectingValueForKey:(NSString *)key {
    if ([key isEqualToString:@"reachable"] || [key isEqualToString:@"reachableViaWWAN"] || [key isEqualToString:@"reachableViaWiFi"]) {
        return [NSSet setWithObject:@"networkReachabilityStatus"];
    }

    return [super keyPathsForValuesAffectingValueForKey:key];
}
```

这个是KVO键值检测方法，当`networkReachabilityStatus`属性变化时，`reachable`、`reachableViaWWAN`和`reachableViaWiFi`这三个属性将收到通知，并更新它们的值。

至此，`AFNetworkReachabilityManager`这个类就分析完了。

## 总结

纵观整个类的代码设计，我们可以学习到很多规范的编码习惯，下面总结一下：

* 当某个类或对象具有不同状态时，应采用NS_ENUM来定义。
* 公开的只读BOOL类型属性，在.h文件通过getter=isXXXX来设定getter方法名称，提高代码的可读性。
* 定义多种初始化方法，并指定一个`Designated Initializer（指定构造方法）`和至少一个`Convenience Initializers(便利性构造方法)`，以适应不同情况下创建对象。
* 类的私有方法可以的话，尽量写成C函数的形式，提高APP的效率。
* 每个定义常量字符串的地方都进行国际化处理
* 合理的使用`#if...#else...#endif`条件编译可以减少编译时间