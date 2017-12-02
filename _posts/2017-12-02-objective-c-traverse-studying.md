---
layout: post
title: Objective-C 遍历方式总结
date: 2017-12-02 10:18:24.000000000 +09:00
tags: iOS
---

在 Objective-C 实际开发过程中，我们经常需要遍历各种 collection，以获得所有元素并进行处理。下面将总结下目前 Objective-C 可用的几种遍历方式以及优缺点，这里的 collection 以 `NSArray`、`NSDictionary` 和 `NSSet` 这几个常用类型来举例。本文的 Demo 下载地址：[**TraverseText**](https://github.com/XuDeHong/iOS-code/tree/master/TraverseText)

## for 循环遍历法

`for 循环遍历`，是 C 语言的语法，是最传统的一种遍历方法。Objective-C 是 C 的超集，自然也继承了这种语言特性，如下面的例子。

```objc
NSLog(@"for循环遍历NSArray:");
for (NSInteger i = 0; i < array.count; i++)
{
    NSLog(@"%@",array[i]);
}
```

```objc
NSLog(@"for循环遍历NSDictionary:");
NSArray *keys = [dict allKeys];
for (NSInteger i = 0; i < keys.count; i++)
{
    id key = keys[i];
    id value = dict[key];
    NSLog(@"key:%@  value:%@",key,value);
}
```

```objc
NSLog(@"for循环遍历NSSet:");
NSArray *objects = [set allObjects];
for (NSInteger i = 0; i < objects.count; i++)
{
    NSLog(@"%@",objects[i]);
}
```

```objc
NSLog(@"for循环反向遍历NSArray:");
for (NSInteger i = array.count - 1; i >= 0; i--)
{
    NSLog(@"%@",array[i]);
}
```

这种老式的遍历方式应该都很熟悉了，要说明的几点如下。

1. 遍历 NSDictionary 时需要先获取所有 `key`，然后遍历 `key` 去获取对应的 `value`；
2. 遍历 NSSet 需要先创建一个中间件 `NSArray`，然后遍历该 NSArray 来获取 NSSet 的对象；
3. 反向遍历在这里只对 NSArray 有意义，因为 NSDictionary 和 NSSet 都是`无序`的。

优点：比较简单，对于 NSArray 能获取到下标。

缺点：对于 NSDictionary 和 NSSet，需要创建一个中间件 NSArray，有额外开销。

## NSEnumerator 遍历法

`NSEnumerator` 是一个`抽象基类`，它定义了下面一个方法和一个属性：

```objc
- (nullable ObjectType)nextObject;	//获取下一个元素
@property (readonly, copy) NSArray<ObjectType> *allObjects;	//获取所有元素
```

NSArray、NSDictionary 和 NSSet 等 collection 类实现了这个方法和继承这个属性，通过生成 NSEnumerator 对象来遍历其中的元素，如下面的例子。

```objc
NSLog(@"NSEnumerator遍历NSArray:");
NSEnumerator *arrayEnumerator = [array objectEnumerator];
id arrayObject;
while ((arrayObject = [arrayEnumerator nextObject]) != nil)
{
    NSLog(@"%@",arrayObject);
}
```

```objc
NSLog(@"NSEnumerator遍历NSDictionary:");
NSEnumerator *dictKeyEnumerator = [dict keyEnumerator];
id key;
while ((key = [dictKeyEnumerator nextObject]) != nil)
{
    id value = dict[key];
    NSLog(@"key:%@  value:%@",key,value);
}
```

```objc
NSLog(@"NSEnumerator遍历NSSet:");
NSEnumerator *setEnumerator = [set objectEnumerator];
id setObject;
while ((setObject = [setEnumerator nextObject]) != nil)
{
    NSLog(@"%@",setObject);
}
```

```objc
NSLog(@"NSEnumerator反向遍历NSArray:");
NSEnumerator *arrayReverseEnumerator = [array reverseObjectEnumerator];
id arrayReverseObject;
while ((arrayReverseObject = [arrayReverseEnumerator nextObject]) != nil)
{
    NSLog(@"%@",arrayReverseObject);
}
```

NSEnumerator 是 `Objective-C 1.0` 时加入的特性，额外的说明如下。

1. 遍历 NSDictionary 时是先获取所有 key 的 enumerator，然后通过 `nextObject` 方法来遍历这个 enumerator，最后获取对应 value；
2. 反向遍历可以通过 `reverseObjectEnumerator` 方法来获取 enumerator，然后遍历这个 enumerator。

优点：遍历各种 collection 的语法较统一，有多种 enumerator 可直接使用。

缺点：需要生成一个 enumerator，有额外开销，对于 NSArray 无法获取下标。

## 快速遍历法

如果某个 collection 类遵从 `NSFastEnumeration`  协议，则说明它支持`快速遍历法`。这个协议定义了下面一个方法：

```objc
- (NSUInteger)countByEnumeratingWithState:(NSFastEnumerationState *)state objects:(id __unsafe_unretained _Nullable [_Nonnull])buffer count:(NSUInteger)len;
```

这个方法的具体原理不阐述，可自行 google。collection 类实现了这个方法后，就可以使用快速遍历法遍历元素，如下面的例子。

```objc
NSLog(@"快速遍历NSArray:");
for (id object in array)
{
    NSLog(@"%@",object);
}
```

```objc
NSLog(@"快速遍历NSDictionary:");
for (id key in dict)
{
    id value = dict[key];
    NSLog(@"key:%@  value:%@",key,value);
}
```

```objc
NSLog(@"快速遍历NSSet:");
for (id object in set)
{
    NSLog(@"%@",object);
}
```

```objc
NSLog(@"快速反向遍历NSArray:");
for (id object in [array reverseObjectEnumerator])
{
    NSLog(@"%@",object);
}
```

快速遍历法是 `Objective-C 2.0` 引入的语言特性，它为 for 循环添加了 `in` 关键字，使得遍历 collection 的语法大大简化，补充下面两点。

1. 跟前面两种遍历方式一样，遍历 NSDictionary 需要先遍历 key，再通过 key 获取value；
2. 反向遍历是通过遍历 `reverseObjectEnumerator` 方法生成的 enumerator。

优点：语法更简洁，无额外开销。

缺点：对于NSArray，无法获取下标。

## 基于 Block 的遍历法

在最新的 Objective-C 语言中，加入了基于 `Block` 的遍历法。这种遍历法让我们对遍历的过程有更大的控制权，先看下面的例子。

```objc
NSLog(@"基于Block遍历NSArray:");
[array enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
    NSLog(@"%@",obj);
}];
```

```objc
NSLog(@"基于Block遍历NSDictionary:");
[dict enumerateKeysAndObjectsUsingBlock:^(id  _Nonnull key, id  _Nonnull obj, BOOL * _Nonnull stop) {
    NSLog(@"key:%@  value:%@",key,obj);
}];
```

```objc
NSLog(@"基于Block遍历NSSet:");
[set enumerateObjectsUsingBlock:^(id  _Nonnull obj, BOOL * _Nonnull stop) {
    NSLog(@"%@",obj);
}];
```

```objc
NSLog(@"基于Block反向遍历NSArray:");
[array enumerateObjectsWithOptions:NSEnumerationReverse usingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
    NSLog(@"%@",obj);
}];
```

在这种基于 Block 的遍历法中，我们需要定义一个 Block，并传递到对应方法的参数中。遍历过程中，每访问一个元素，就会执行一次 Block 里面的代码，另外补充说明如下。

1. 对于 NSArray，遍历的时候既能获取下标，又能获取对应的元素；而对于 NSDictionary，遍历的时候 key 和 value 能同时获取；
2. Block 有一个参数 `stop`，通过设置该参数为 YES，可以结束遍历过程；
3. 反向遍历需要通过设置 `NSEnumerationOptions` 类型的参数为 `NSEnumerationReverse`，这是一个枚举类型，我们可以通过“`按位或`”来将不同类型组合在一起，表明不同的遍历方式；
4. 这种遍历法使用 `GCD` 来实现`并发`遍历。

优点：无额外开销，能够对遍历过程和方式进行控制。

缺点：与快速遍历法相比，没那么简洁。