---
layout: post
title: 绕过苹果审核使用动态库
---

##如何绕过
关键就是要躲过苹果的api检测。苹果至少有2种方式对api进行检测：

####绕过静态检测

 - 利用字符串加密拼接成函数名，然后动态调用的方式
 - 从服务器下载到函数名，然后动态调用的方式

####绕过运行时检测

 - 审核时不出现调用动态库的模块

本文就是通过从服务器下载函数名的方式绕过静态检测。

##步骤
 先附上[demo](https://github.com/pingyourid/DLLoad)，这里面的动态库由于时间原因只支持了x86的cpu架构（可以在5s以上模拟器上跑，太懒了不想合并）

- 动态库代码

```
 @interface Person : NSObject
 - (void)run; 
 @end
 
 @implementation Person 
 -(void)run { 
 	NSLog(@"let's run."
 	); 
    UIAlertView *alert = [[UIAlertView alloc] initWithTitle:@"The Second Alert" message:nil delegate:nil cancelButtonTitle:nil otherButtonTitles:@"done", nil]; 
 	[alert show]; 
 } 
 @end
```

- 服务器需要准备动态库和执行本地代码的js文件,这边模拟了下，存到了document目录下

```   
	//模拟下载js到沙盒
    NSString *jsPath = [self copyRes:@"load.js"];
    //模拟下载动态库到沙盒
    NSString *dlPath = [self copyRes:@"Dylib.framework"];
```

- 传入动态文件库到js代码内，并执行js

```
	[JPEngine startEngine];

    NSError *error = nil;
    NSString *script = [NSString stringWithContentsOfFile:jsPath encoding:NSUTF8StringEncoding error:&error];
    
    script = [script stringByReplacingOccurrencesOfString:@"aaaaa" withString:dlPath];
    
    if (script) {
        [JPEngine evaluateScript:script];
    }
```

这里的startEngine是[JSPatch](https://github.com/bang590/JSPatch)库中的方法，[博客传送门](http://blog.cnbang.net/works/2767/)，JSPatch利用JavaScriptCore.framework在js中注册oc的代码，回调到oc时通过runtime把传入的字符串变成方法后调用。

 这边就是要在js中写一些被苹果禁止的方法，通过runtime在oc中调用。

- js替换掉原来的空方法

 js:替换JSLoad类的loadDl方法
 
```
 defineClass('JSLoad', null, {
    loadDl: function() {
        var bundle = require('NSBundle').bundleWithPath('aaaaa');
        bundle.loadAndReturnError(null);
    }, 
 });
```
 
 oc:预埋的空方法
 
```
 + (void)loadDl
 {

 }
```

- 执行已经被替换掉的方法，该方法会去调用加载动态库

```
[JSLoad loadDl];
```

```
var bundle = require('NSBundle').bundleWithPath('aaaaa');
     bundle.loadAndReturnError(null);
```

- 使用动态库中的类

```
Class rootClass = NSClassFromString(@"Person");
    if (rootClass) {
        id object = [[rootClass alloc] init];
        [object run];
    }
```

loadAndReturnError就是避免要被苹果检测到的函数，如此这般，就完成了从服务器下载到函数名，然后动态调用的方式。

##总结

这种方式可以百分百的绕过苹果的审核，但是听说appstore上的应用使用沙盒内的动态库时，还会有个签名，所以demo中可能能成功的调用，上架后不一定能成功（所以这可能并没有什么卵用）,所以这里还需要试验一下。

但是能确定的是，通过JSPatch的方式可以轻松的在appstore中使用私有api。



