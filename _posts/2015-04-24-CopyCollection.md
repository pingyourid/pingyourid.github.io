---
layout: post
title: ios容器类内存管理详解
---

![_config.yml](/images/copy_type.jpg)
## 浅拷贝

有多种方法来创建一个集合（a collection）的浅拷贝。当你创建一个浅拷贝，便给原来`集合的对象`发送一个 retain 消息，对象的指针被复制到新的集合。清单1显示了一些使用浅拷贝来创建新的集合的方法。

- 清单 1  创建浅拷贝

{% highlight ruby linenos %}
NSArray *shallowCopyArray=[someArray copyWithZone:nil];

NSDictionary *shallowCopyDict=[[NSDictionary alloc] initWithDictionary: someDictionary copyItems: NO];
{% endhighlight %}

这些技巧不局限于上述集合类。比如，您可以使用 copyWithZone: 方法拷贝一个集合（a set），或者使用 mutableCopyWithZone:，也可以使用 initWithArray:copyItems: 方法来拷贝数组。

## 深拷贝

有两种方法来创建一个集合的深拷贝。您可以使用集合的等价方法 initWithArray:copyItems:，不过第2个参数为：YES。如果您使用这种方法创建了一个集合的深拷贝，集合中的每个对象都被发送一个 copyWithZone: 消息。`如果集合中的对象是不可变的，不会产生新的地址（解释见下文），如果集合中的对象都实现 NSCopying 协议，对象被深拷贝到新的集合，这个集合是新复制的对象的唯一所有者。如果对象没有实现 NSCopying 协议，那么试图用这种方法复制它们会导致一个运行时错误结果`。然而，copyWithZone: 产生一个浅拷贝。这种复制方法仅产生一个`一级深度的拷贝`。如果你只需要一级深度的拷贝，您可以使用清单2的方法

- 清单 2  创建深拷贝

{% highlight ruby linenos %}
NSArray *deepCopyArray=[[NSArray alloc] initWithArray: someArray copyItems: YES];
{% endhighlight %}

这个技巧也适用于其他集合。使用集合的等价方法 initWithArray:copyItems:，第2个参数为：YES。

如果您需要一个真正的深拷贝，比如对于一个数组，可以使用archive和unchive来处理，数组元素要符合 NSCoding 协议。您可以使用清单3的方法。

- 清单 3  真正的深拷贝
{% highlight ruby linenos %}
NSArray* trueDeepCopyArray = [NSKeyedUnarchiver unarchiveObjectWithData: [NSKeyedArchiver archivedDataWithRootObject: oldArray]];
{% endhighlight %}

####举例
{% highlight ruby linenos %}
 NSArray *n = [[NSArray alloc] initWithObjects:[NSMutableString stringWithString:@"123"],@"456", nil];
 
    // = [n copy]
    // The copy returned is immutable if the consideration “immutable vs. mutable” applies to the receiving object;
    // otherwise the exact nature of the copy is determined by the class.
    NSMutableArray *n1 = [n copyWithZone:nil];
 
    // = [n mutableCopy]
    // The copy returned is mutable whether the original is mutable or not.
    NSMutableArray *n2  = [n mutableCopyWithZone:nil];
 
    NSArray *n3 = [[NSArray alloc] initWithArray:n copyItems:NO];
 
    NSArray *n4 = [[NSArray alloc] initWithArray:n copyItems:YES];
 
    NSArray *n5 = [NSKeyedUnarchiver unarchiveObjectWithData:[NSKeyedArchiver archivedDataWithRootObject: n]];
 
    NSLog(@"Array n = %@", n);
    NSLog(@"Array n1 = %@", n1);
    NSLog(@"Array n2 = %@", n2);
    NSLog(@"Array n3 = %@", n3);
    NSLog(@"Array n4 = %@", n4);
    NSLog(@"Array n5 = %@", n5);
 
    NSMutableString *nstr1 = [n objectAtIndex:0];
    [nstr1 appendString:@"abc"];
 
    [n2 addObject:@"xxx"];
 
    NSLog(@"Array n = %@", n);
    NSLog(@"Array n1 = %@", n1);
    NSLog(@"Array n2 = %@", n2);
    NSLog(@"Array n3 = %@", n3);
    NSLog(@"Array n4 = %@", n4);
    NSLog(@"Array n5 = %@", n5); 
{% endhighlight %}

![_config.yml](/images/copy1.jpg)

##可变容器和不可变容器

- NSArray对象不可变
- NSMutableArray对象可变
- `@[]或者@{}对象是不可变对象`

##copy与mutableCopy
- copy会返回一个不可变对象

`对不可变对象使用copy,会返回原对象，不会开辟新的内存（可能是因为系统认为都是用不可变对象，没必要重新开辟一个空间）。`

`对不可变对象使用mutableCopy,会开辟新内存，返回一个可变对象。`

- mutableCopy会返回一个可变对象

`对可变对象使用copy,会开辟新内存，返回一个不可变对象。`

`对可变对象使用mutableCopy,会开辟新内存，返回一个可变对象。`

####举例

{% highlight ruby linenos %}
NSArray *n = [[NSMutableArray alloc] initWithObjects:[NSMutableString stringWithString:@"123"],@"456", nil];
    NSLog(@"%p",n);
   
    NSArray *n1 = [n copy];
    if ([n1 isKindOfClass:[NSArray class]]) {
        NSLog(@"1");
    }
    else
    {
        NSLog(@"1-");
    }
    NSLog(@"%p",n1);
   
    NSArray *n2 = [n mutableCopy];
    if ([n2 isKindOfClass:[NSMutableArray class]]) {
        NSLog(@"2");
    }
    else
    {
        NSLog(@"2-");
    }
    NSLog(@"%p",n2);
{% endhighlight %}

![_config.yml](/images/copy2.jpg)

{% highlight ruby linenos %}
NSArray *n = [[NSArray alloc] initWithObjects:[NSMutableString stringWithString:@"123"],@"456", nil];
    NSLog(@"%p",n);
   
    NSArray *n1 = [n copy];
    if ([n1 isKindOfClass:[NSArray class]]) {
        NSLog(@"1");
    }
    else
    {
        NSLog(@"1-");
    }
    NSLog(@"%p",n1);
   
    NSArray *n2 = [n mutableCopy];
    if ([n2 isKindOfClass:[NSMutableArray class]]) {
        NSLog(@"2");
    }
    else
    {
        NSLog(@"2-");
    }
    NSLog(@"%p",n2);
{% endhighlight %}

![_config.yml](/images/copy3.jpg)

##防止设置空值crash

- 会crash的调用

setObject..forkey

addObject

dic[**] = nil

dic = @{@"**",nil}

- 不会crash的调用

[NSDictionary dic**objectandkeys:nil,@“fa”]

setValue..forkey

- 建议

所以好的框架应当`重写setObject..forkey,addObject方法`，避免crash。

`减少使用简写的赋值，初始化操作dic[aa] = nil，dic = @{@"**",nil}`

可以适当的使用@[].mutableCopy,@{}.mutableCopy来初始化可变对象，或者取值以简化代码。

##多线程下对容器的使用

对一个容器枚举时，如果改变容器里的内容，会crash。所以`尽量要枚举一个不可变的容器`。也就是说对一个可变容器枚举前，可以 A = 可变容器.copy后枚举A。

同理，多线程下，[NSMutableArray array witharray:(一个mutablearry B)]也可能会因为B的改变而crash(初始化也是个枚举过程，NSMutableArray是非线程安全的)。所以这里也需要B.copy后再传值。

其他Seria,Archive类操作容器时，也应当`确保操作的是一个不可变对象`。

##总结

- 应当`重写setObject..forkey,addObject方法`，避免crash。
- `减少使用简写的赋值，初始化操作dic[aa] = nil，dic = @{@"**",nil}`
- 可以适当的使用@[].mutableCopy,@{}.mutableCopy来初始化可变对象，或者取值以简化代码。
- 尽量要枚举一个不可变的容器

######参考文档

[iOS学习笔记——拷贝集合类（Copying Collections）](http://zgia.net/?p=299)

[苹果官方文档](https://developer.apple.com/library/mac/documentation/cocoa/conceptual/Collections/Articles/Copying.html#//apple_ref/doc/uid/TP40010162-SW3)
