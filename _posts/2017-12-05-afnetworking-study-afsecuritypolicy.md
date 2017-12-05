---
layout: post
title: AFNetworking 3.1.0 源码学习（二）AFSecurityPolicy
date: 2017-12-05 16:30:24.000000000 +09:00
tags: iOS

---

本文基于[AFNetworking 3.1.0](https://github.com/AFNetworking/AFNetworking)进行学习，我们这次学习`AFSecurityPolicy`，这是一个用于`认证服务器返回的证书`的模块，读者可以打开源代码跟着文章思路来学习。

## 头文件部分

先来看看AFSecurityPolicy.h 文件

``` objc
#import <Foundation/Foundation.h>
#import <Security/Security.h>

typedef NS_ENUM(NSUInteger, AFSSLPinningMode) {
    AFSSLPinningModeNone,   //无条件信任服务器的证书
    AFSSLPinningModePublicKey, //通过服务器返回的证书中的PublickKey进行认证
    AFSSLPinningModeCertificate,    //通过服务器返回的证书进行认证
};
```

系统的Security就是用来做证书认证的，然后是一个关于认证模式枚举值的定义。这里说明一下，服务器返回的证书中包含有公钥PublicKey，如果认证通过，则APP传输数据时则用该PublicKey进行加密数据。AFSSLPinningModePublicKey是只对证书中的PublicKey进行认证，而AFSSLPinningModeCertificate是对整个证书内容认证。

``` objc
//当前认证模式
@property (readonly, nonatomic, assign) AFSSLPinningMode SSLPinningMode;
//当前持有的证书，这里是本地的证书，不是当前认证服务器返回的证书，后面会讲到
@property (nonatomic, strong, nullable) NSSet <NSData *> *pinnedCertificates;
//是否允许无效或过期证书
@property (nonatomic, assign) BOOL allowInvalidCertificates;
//是否认证证书中的域名
@property (nonatomic, assign) BOOL validatesDomainName;
```

这是对外公开的属性，源代码的注释都说得很清楚了。这里要注意的是，pinnedCertificates里的证书只要有一个认证通过，则evaluateServerTrust:forDomain: 方法返回true，这个后面会讲到。

```objc
+ (NSSet <NSData *> *)certificatesInBundle:(NSBundle *)bundle;
```

返回指定bundle中的证书集合。

```objc
+ (instancetype)defaultPolicy;
```

返回默认的实例对象，默认的认证配置为：不允许无效或过期的证书；认证证书中的域名；不对证书和证书中的公钥进行认证。

```objc
+ (instancetype)policyWithPinningMode:(AFSSLPinningMode)pinningMode;

+ (instancetype)policyWithPinningMode:(AFSSLPinningMode)pinningMode withPinnedCertificates:(NSSet <NSData *> *)pinnedCertificates;
```

两个实例初始化方法，前者通过指定认证模式来实例化，后者通过指定认证模式和证书集合来实例化。

``` objc
- (BOOL)evaluateServerTrust:(SecTrustRef)serverTrust
                  forDomain:(nullable NSString *)domain;
```

核心方法，外界通过调用这个方法来进行认证，认证通过则返回true，否则返回false。这里出现了一个SecTrustRef类型的参数，目前先这么理解，它包含了认证过程所需要的数据，包括认证的证书和认证的策略，后面会看到认证过程都是在这个变量上进行的。

AFSecurityPolicy.h 学习完毕，接下来看AFSecurityPolicy.m文件。

## 实现文件部分

实现文件我们先看看这个函数

```objc
//从证书中获取公钥
static id AFPublicKeyForCertificate(NSData *certificate) {
    id allowedPublicKey = nil;	//被认证的公钥
    SecCertificateRef allowedCertificate;	//被认证的证书
    SecPolicyRef policy = nil;	//认证的策略
    SecTrustRef allowedTrust = nil;		//包含认证需要的数据的变量
    SecTrustResultType result;	//认证的结果类型
	//将NSData类型的证书转换成SecCertificateRef类型的证书
    allowedCertificate = SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certificate);
  	//判断allowedCertificate是否为空，如果为空，则直接跳到_out标记的代码，否则继续
    __Require_Quiet(allowedCertificate != NULL, _out);
	//初始化policy，认证策略采用X.509数字证书标准
    policy = SecPolicyCreateBasicX509();
  	//通过allowedCertificate证书、policy认证证书标准来初始化allowedTrust变量，如果出错，则跳到_out标记
    //否则继续
    __Require_noErr_Quiet(SecTrustCreateWithCertificates(allowedCertificate, policy, &allowedTrust), _out);
  	//认证的过程，如果出错，则跳到_out标记，否则将认证结果保存到result变量
    __Require_noErr_Quiet(SecTrustEvaluate(allowedTrust, &result), _out);
	//从allowedTrust中获取公钥
    allowedPublicKey = (__bridge_transfer id)SecTrustCopyPublicKey(allowedTrust);
	//_out标记代码，都是释放CoreFundation对象的代码
_out:
    if (allowedTrust) {
        CFRelease(allowedTrust);
    }

    if (policy) {
        CFRelease(policy);
    }

    if (allowedCertificate) {
        CFRelease(allowedCertificate);
    }
	//返回公钥
    return allowedPublicKey;
}
```

这个函数是用于从指定的证书中获取公钥的，大部分解释都在上面的注释里，这个过程大致了解一下就行，这里补充几点。

1. \__bridge和\_\_bridge_transfer是用于`Objective-C对象`和`CoreFoudation对象`的转换，这是一种叫做`无缝桥接（Toll-Free Bridging）`的技术。 除此之外，还有一个关键字是\_\_bridge\_retained。它们三者的区别如下：

   * __bridge：只做类型转换，但是不修改对象（内存）管理权；
   * __bridge_transfer：与CFBridgingRelease相同，将Core Foundation的对象转换为Objective-C的对象，同时将对象（内存）的管理权交给ARC；
   * __bridge_retained：与CFBridgingRetain相同，将Objective-C的对象转换为Core Foundation的对象，同时将对象（内存）的管理权交给我们，后续需要使用CFRelease或者相关方法来释放对象。

2. CoreFoudation对象不在ARC内存管理范围内，需要自己手动释放内存，CFRelease()就是用于做释放操作；

3. \__Require_Quiet和\_\_Require_noErr_Quiet是两个宏定义，其定义分别如下：

   ```objc
   	#define __Require_Quiet(assertion, exceptionLabel)                            \
   	  do                                                                          \
   	  {                                                                           \
   		  if ( __builtin_expect(!(assertion), 0) )                                \
   		  {                                                                       \
   			  goto exceptionLabel;                                                \
   		  }                                                                       \
   	  } while ( 0 )
   	  
   	  #define __Require_noErr_Quiet(errorCode, exceptionLabel)                    \
   	  do                                                                          \
   	  {                                                                           \
   		  if ( __builtin_expect(0 != (errorCode), 0) )                            \
   		  {                                                                       \
   			  goto exceptionLabel;                                                \
   		  }                                                                       \
   	  } while ( 0 )
   ```

   前者当条件返回false时跳到exceptionLabel标记的代码，后者当条件抛出异常时跳到exceptionLabel标记的代码。


```objc
#if !TARGET_OS_IOS && !TARGET_OS_WATCH && !TARGET_OS_TV		//如果是macOS系统则定义该方法
//将SecKeyRef类型的公钥key转换成NSData类型
static NSData * AFSecKeyGetData(SecKeyRef key) {
    CFDataRef data = NULL;

    __Require_noErr_Quiet(SecItemExport(key, kSecFormatUnknown, kSecItemPemArmour, NULL, &data), _out);

    return (__bridge_transfer NSData *)data;

_out:
    if (data) {
        CFRelease(data);
    }

    return nil;
}
#endif
```

```objc
//比较两个公钥key是否相同
static BOOL AFSecKeyIsEqualToKey(SecKeyRef key1, SecKeyRef key2) {
#if TARGET_OS_IOS || TARGET_OS_WATCH || TARGET_OS_TV
    return [(__bridge id)key1 isEqual:(__bridge id)key2];
#else
    return [AFSecKeyGetData(key1) isEqual:AFSecKeyGetData(key2)];
#endif
}
```

这是一个用于比较两个key是否相同的方法。如果是macOS系统，则用上面的方法将key转换为NSData类型再比较，否则转换为id类型再比较。

```objc
//从SecTrustRef类型对象中获取所有证书
static NSArray * AFCertificateTrustChainForServerTrust(SecTrustRef serverTrust) {
    CFIndex certificateCount = SecTrustGetCertificateCount(serverTrust);
    NSMutableArray *trustChain = [NSMutableArray arrayWithCapacity:(NSUInteger)certificateCount];

    for (CFIndex i = 0; i < certificateCount; i++) {
        SecCertificateRef certificate = SecTrustGetCertificateAtIndex(serverTrust, i);
        [trustChain addObject:(__bridge_transfer NSData *)SecCertificateCopyData(certificate)];
    }

    return [NSArray arrayWithArray:trustChain];
}
```

```objc
//从SecTrustRef类型对象中获取所有公钥key
static NSArray * AFPublicKeyTrustChainForServerTrust(SecTrustRef serverTrust) {
  	//初始化policy，认证策略采用X.509数字证书标准
    SecPolicyRef policy = SecPolicyCreateBasicX509();
  	//从serverTrust对象获取证书的数目
    CFIndex certificateCount = SecTrustGetCertificateCount(serverTrust);
  	//根据证书的数目初始化一个数组
    NSMutableArray *trustChain = [NSMutableArray arrayWithCapacity:(NSUInteger)certificateCount];
    for (CFIndex i = 0; i < certificateCount; i++) {
      	//根据下标和serverTrust获取证书
        SecCertificateRef certificate = SecTrustGetCertificateAtIndex(serverTrust, i);
		//将SecCertificateRef证书对象放到数组里
        SecCertificateRef someCertificates[] = {certificate};
      	//根据someCertificates创建CFArrayRef数组
        CFArrayRef certificates = CFArrayCreate(NULL, (const void **)someCertificates, 1, NULL);
		//根据certificates和policy初始化一个新的SecTrustRef对象
      	//这里为什么从serverTrust拿出了证书certificate，然后又创建一个新的trust呢？我觉得是想把policy联		 //系起来，serverTrust可能并没有设置认证策略，所以需要创建一个新的trust
        SecTrustRef trust;
        __Require_noErr_Quiet(SecTrustCreateWithCertificates(certificates, policy, &trust), _out);
		//认证刚刚初始化的trust，并将结果赋值到result
        SecTrustResultType result;
        __Require_noErr_Quiet(SecTrustEvaluate(trust, &result), _out);
		//从trust获取公钥key，并加入到数组中
        [trustChain addObject:(__bridge_transfer id)SecTrustCopyPublicKey(trust)];
	//_out标记代码，都是释放CoreFundation对象的代码
    _out:
        if (trust) {
            CFRelease(trust);
        }

        if (certificates) {
            CFRelease(certificates);
        }

        continue;
    }
    CFRelease(policy);
	//返回公钥key的数组
    return [NSArray arrayWithArray:trustChain];
}
```

```objc
@interface AFSecurityPolicy()
//与.h的SSLPinningMode对应，当前认证模式
@property (readwrite, nonatomic, assign) AFSSLPinningMode SSLPinningMode;
//保存公钥key的集合，这里是本地证书的公钥key，不是认证服务器返回的公钥key
@property (readwrite, nonatomic, strong) NSSet *pinnedPublicKeys;
@end
```

```objc
//获取指定bundle下的所有证书
+ (NSSet *)certificatesInBundle:(NSBundle *)bundle {
    NSArray *paths = [bundle pathsForResourcesOfType:@"cer" inDirectory:@"."];

    NSMutableSet *certificates = [NSMutableSet setWithCapacity:[paths count]];
    for (NSString *path in paths) {
        NSData *certificateData = [NSData dataWithContentsOfFile:path];
        [certificates addObject:certificateData];
    }

    return [NSSet setWithSet:certificates];
}
```

```objc
//返回默认的证书，与上一个方法配合，返回当前ipa包目录下的所有证书
+ (NSSet *)defaultPinnedCertificates {
    static NSSet *_defaultPinnedCertificates = nil;
    static dispatch_once_t onceToken;
  	//这里的block代码只执行一次
    dispatch_once(&onceToken, ^{
        NSBundle *bundle = [NSBundle bundleForClass:[self class]];
        _defaultPinnedCertificates = [self certificatesInBundle:bundle];
    });

    return _defaultPinnedCertificates;
}
```

```objc
//返回默认的实例，默认SSLPinningMode为AFSSLPinningModeNone
+ (instancetype)defaultPolicy {
    AFSecurityPolicy *securityPolicy = [[self alloc] init];
    securityPolicy.SSLPinningMode = AFSSLPinningModeNone;

    return securityPolicy;
}
//指定AFSSLPinningMode来实例化
+ (instancetype)policyWithPinningMode:(AFSSLPinningMode)pinningMode {
    return [self policyWithPinningMode:pinningMode withPinnedCertificates:[self defaultPinnedCertificates]];
}
//指定AFSSLPinningMode和证书来实例化
+ (instancetype)policyWithPinningMode:(AFSSLPinningMode)pinningMode withPinnedCertificates:(NSSet *)pinnedCertificates {
    AFSecurityPolicy *securityPolicy = [[self alloc] init];
    securityPolicy.SSLPinningMode = pinningMode;

    [securityPolicy setPinnedCertificates:pinnedCertificates];

    return securityPolicy;
}
//init方法，只设置了validatesDomainName为YES，认证证书的域名
- (instancetype)init {
    self = [super init];
    if (!self) {
        return nil;
    }

    self.validatesDomainName = YES;

    return self;
}
```

这里是几个初始化实例的方法。

```objc
//pinnedCertificates的setter，每次设置证书时，都同时更新pinnedPublicKeys
- (void)setPinnedCertificates:(NSSet *)pinnedCertificates {
    _pinnedCertificates = pinnedCertificates;

    if (self.pinnedCertificates) {
        NSMutableSet *mutablePinnedPublicKeys = [NSMutableSet setWithCapacity:[self.pinnedCertificates count]];
      	//通过上面的私有方法，从证书中获取公钥key，然后保存在pinnedPublicKeys
        for (NSData *certificate in self.pinnedCertificates) {
            id publicKey = AFPublicKeyForCertificate(certificate);
            if (!publicKey) {
                continue;
            }
            [mutablePinnedPublicKeys addObject:publicKey];
        }
        self.pinnedPublicKeys = [NSSet setWithSet:mutablePinnedPublicKeys];
    } else {
        self.pinnedPublicKeys = nil;
    }
}
```

接下来是整个类的最核心方法

```objc
//认证的核心方法
- (BOOL)evaluateServerTrust:(SecTrustRef)serverTrust
                  forDomain:(NSString *)domain
{
    //这里的if大致意思是如果没有证书，或者SSLPinningMode为AFSSLPinningModeNone，那么认证域名就没有意义
    if (domain && self.allowInvalidCertificates && self.validatesDomainName && (self.SSLPinningMode == AFSSLPinningModeNone || [self.pinnedCertificates count] == 0)) {
        // https://developer.apple.com/library/mac/documentation/NetworkingInternet/Conceptual/NetworkingTopics/Articles/OverridingSSLChainValidationCorrectly.html
        //  According to the docs, you should only trust your provided certs for evaluation.
        //  Pinned certificates are added to the trust. Without pinned certificates,
        //  there is nothing to evaluate against.
        //
        //  From Apple Docs:
        //          "Do not implicitly trust self-signed certificates as anchors (kSecTrustOptionImplicitAnchors).
        //           Instead, add your own (self-signed) CA certificate to the list of trusted anchors."
        NSLog(@"In order to validate a domain name for self signed certificates, you MUST use pinning.");
        return NO;
    }

    NSMutableArray *policies = [NSMutableArray array];
    if (self.validatesDomainName) {
      	//如果认证域名，则根据domain生成一个认证SSL的SecPolicy策略对象，并添加到数组中
        [policies addObject:(__bridge_transfer id)SecPolicyCreateSSL(true, (__bridge CFStringRef)domain)];
    } else {
      	//如果不认证域名，则生成一个认证根证书的SecPolicy策略对象，并添加到数组中
        [policies addObject:(__bridge_transfer id)SecPolicyCreateBasicX509()];
    }
	//将serverTrust与policies关联起来
    SecTrustSetPolicies(serverTrust, (__bridge CFArrayRef)policies);

    if (self.SSLPinningMode == AFSSLPinningModeNone) {
      	//如果是AFSSLPinningModeNone，则看是否允许无效或过期证书，又或者serverTrust是否认证通过
        return self.allowInvalidCertificates || AFServerTrustIsValid(serverTrust);
    } else if (!AFServerTrustIsValid(serverTrust) && !self.allowInvalidCertificates) {
      	//如果serverTrust认证不通过，并且不允许无效或过期证书，则返回NO
        return NO;
    }

    switch (self.SSLPinningMode) {
        case AFSSLPinningModeNone:
        default:
            return NO;
        //认证证书和公钥key
        case AFSSLPinningModeCertificate: {
            NSMutableArray *pinnedCertificates = [NSMutableArray array];
          	//取出本地证书
            for (NSData *certificateData in self.pinnedCertificates) {
                [pinnedCertificates addObject:(__bridge_transfer id)SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certificateData)];
            }
          	//将serverTrust与本地证书关联起来
            SecTrustSetAnchorCertificates(serverTrust, (__bridge CFArrayRef)pinnedCertificates);
			//认证是否通过
            if (!AFServerTrustIsValid(serverTrust)) {
                return NO;
            }

            // obtain the chain after being validated, which *should* contain the pinned certificate in the last position (if it's the Root CA)
          	//如果认证通过，则从serverTrust获取服务器返回的证书
            NSArray *serverCertificates = AFCertificateTrustChainForServerTrust(serverTrust);
            //如果本地证书包含有服务器返回的证书，则认证过程通过并结束
          	//这里不太明白，AFServerTrustIsValid()方法如果已经认证通过了，但本地证书不包含服务器证书就代       			 //表认证不通过？
            for (NSData *trustChainCertificate in [serverCertificates reverseObjectEnumerator]) {
                if ([self.pinnedCertificates containsObject:trustChainCertificate]) {
                    return YES;
                }
            }
            
            return NO;
        }
        case AFSSLPinningModePublicKey: {
            NSUInteger trustedPublicKeyCount = 0;
          	//从serverTrust获取公钥key(从服务器返回的)
            NSArray *publicKeys = AFPublicKeyTrustChainForServerTrust(serverTrust);
			//判断本地的公钥key是否包含服务器返回的公钥key
            for (id trustChainPublicKey in publicKeys) {
                for (id pinnedPublicKey in self.pinnedPublicKeys) {
                    if (AFSecKeyIsEqualToKey((__bridge SecKeyRef)trustChainPublicKey, (__bridge SecKeyRef)pinnedPublicKey)) {
                        trustedPublicKeyCount += 1;
                    }
                }
            }
            return trustedPublicKeyCount > 0;
        }
    }
    
    return NO;
}
```

```objc
#pragma mark - NSKeyValueObserving
//KVO方法，当pinnedCertificates属性变更时，通知pinnedPublicKeys对象
+ (NSSet *)keyPathsForValuesAffectingPinnedPublicKeys {
    return [NSSet setWithObject:@"pinnedCertificates"];
}

#pragma mark - NSSecureCoding
//NSSecureCoding协议方法，用于对象编码和解码。
//至于NSSecureCoding和NSCoding的区别可以看这篇文章http://nshipster.cn/nssecurecoding/
+ (BOOL)supportsSecureCoding {
    return YES;
}

- (instancetype)initWithCoder:(NSCoder *)decoder {

    self = [self init];
    if (!self) {
        return nil;
    }

    self.SSLPinningMode = [[decoder decodeObjectOfClass:[NSNumber class] forKey:NSStringFromSelector(@selector(SSLPinningMode))] unsignedIntegerValue];
    self.allowInvalidCertificates = [decoder decodeBoolForKey:NSStringFromSelector(@selector(allowInvalidCertificates))];
    self.validatesDomainName = [decoder decodeBoolForKey:NSStringFromSelector(@selector(validatesDomainName))];
    self.pinnedCertificates = [decoder decodeObjectOfClass:[NSArray class] forKey:NSStringFromSelector(@selector(pinnedCertificates))];

    return self;
}

- (void)encodeWithCoder:(NSCoder *)coder {
    [coder encodeObject:[NSNumber numberWithUnsignedInteger:self.SSLPinningMode] forKey:NSStringFromSelector(@selector(SSLPinningMode))];
    [coder encodeBool:self.allowInvalidCertificates forKey:NSStringFromSelector(@selector(allowInvalidCertificates))];
    [coder encodeBool:self.validatesDomainName forKey:NSStringFromSelector(@selector(validatesDomainName))];
    [coder encodeObject:self.pinnedCertificates forKey:NSStringFromSelector(@selector(pinnedCertificates))];
}

#pragma mark - NSCopying
//NSCopying协议方法
- (instancetype)copyWithZone:(NSZone *)zone {
    AFSecurityPolicy *securityPolicy = [[[self class] allocWithZone:zone] init];
    securityPolicy.SSLPinningMode = self.SSLPinningMode;
    securityPolicy.allowInvalidCertificates = self.allowInvalidCertificates;
    securityPolicy.validatesDomainName = self.validatesDomainName;
    securityPolicy.pinnedCertificates = [self.pinnedCertificates copyWithZone:zone];

    return securityPolicy;
}
```

这个模块的代码虽然不长，但如果没有网络安全的基础知识，看着会很吃力。总的来说，主要是对`SecTrustRef`这个类型对象的操作，这个对象包含了`SecCertificateRef`证书，`SecKeyRef`公钥和`SecPolicyRef`认证策略，认证的结果是`SecTrustResultType`类型变量。