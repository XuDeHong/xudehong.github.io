---
layout: post
title: Objective-C 单例模式的实现
date: 2019-03-19 09:45:24.000000000 +09:00
tags: iOS
---

```objective-c
//
//  Singleton.h
//  SingletonText
//
//  Created by 许德鸿 on 2019/3/19.
//  Copyright © 2019 XuDeHong. All rights reserved.
//

#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface Singleton : NSObject

+ (instancetype)sharedInstance;

@end

NS_ASSUME_NONNULL_END
```

```
//
//  Singleton.m
//  SingletonText
//
//  Created by 许德鸿 on 2019/3/19.
//  Copyright © 2019 XuDeHong. All rights reserved.
//

#import "Singleton.h"

static Singleton *sharedInstanceObject = nil;

@implementation Singleton

+(instancetype)allocWithZone:(struct _NSZone *)zone
{
    if(!sharedInstanceObject)
    {
        static dispatch_once_t onceToken;
        dispatch_once(&onceToken, ^{
            sharedInstanceObject = [super allocWithZone:zone];
        });
    }
    return sharedInstanceObject;
}

- (instancetype)init
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstanceObject = [super init];
    });
    return sharedInstanceObject;
}

- (id)copyWithZone:(NSZone *)zone
{
    return sharedInstanceObject;
}

- (id)mutableCopyWithZone:(NSZone *)zone
{
    return sharedInstanceObject;
}

+ (instancetype)sharedInstance
{
    return [[self alloc] init];
}

@end
```