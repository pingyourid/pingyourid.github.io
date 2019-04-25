---
layout: post
title: 保存app内容到手机桌面
---

今天，我发现淘宝手机app可以把用户喜欢的店铺保存到app的桌面上，感觉很神奇，研究了下怎么做，并记录下来顺便分享下心得。附上[demo地址](https://github.com/pingyourid/AppWebClip)

下面是实际效果:

安装描述文件

![_config.yml](/images/profile.gif)

safari生成webclip

![_config.yml](/images/safari1.gif)


这种效果就是苹果的webclip,app上要生成它主要有2种方式。

## 通过安装描述文件的方式生成webclip
- 使用iphone configuration utility生成一个webclip描述文件。

	下载iphone configuration utility后配置一个描述文件，导出即可。
- 使用safari安装这个描述文件。

	使用`[UIApplication sharedApplication] openURL:`的方式无法直接打开描述文件，UIWebView也不支持打开这种文件类型。
    
	safari是可以直接安装描述文件的，但是safari和应用是2个独立的沙盒，所以这里需要解决应用和safari共享文件的问题。这里使用的思路是把app作为一个服务器，让safari访问这个服务器获取到描述文件进行安装，因为程序进入后台后还可以运行一段时间，所以这里是可行的。

	可以使用第三方库[CocoaHTTPServer](https://github.com/robbiehanson/CocoaHTTPServer)在app端运行一个服务器。safari中访问 loacalhost:端口号/目录即可打开文件。
{% highlight ruby linenos %}
- (void)startServer
{
    // Create server using our custom MyHTTPServer class
    _httpServer = [[RoutingHTTPServer alloc] init];
    
    // Tell the server to broadcast its presence via Bonjour.
    // This allows browsers such as Safari to automatically discover our service.
    [_httpServer setType:@"_http._tcp."];
    
    // Normally there's no need to run our server on any specific port.
    // Technologies like Bonjour allow clients to dynamically discover the server's port at runtime.
    // However, for easy testing you may want force a certain port so you can just hit the refresh button.
    [_httpServer setPort:12345];

    
    NSString *documentsDirectory = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) objectAtIndex:0];
    [_httpServer setDocumentRoot:documentsDirectory];
    
    if (_httpServer.isRunning) [_httpServer stop];
    
    NSError *error;
    if([_httpServer start:&error])
    {
        NSLog(@"Started HTTP Server on port %hu", [_httpServer listeningPort]);
    }
    else
    {
        NSLog(@"Error starting HTTP Server: %@", error);
        // Probably should add an escape - but in practice never loops more than twice (bug filed on GitHub https://github.com/robbiehanson/CocoaHTTPServer/issues/88)
        [self startServer];
    }
}

- (void)applicationDidEnterBackground:(UIApplication *)application {
    // Use this method to release shared resources, save user data, invalidate timers, and store enough application state information to restore your application to its current state in case it is terminated later.
    // If your application supports background execution, this method is called instead of applicationWillTerminate: when the user quits.
    backgroundTask = [application beginBackgroundTaskWithExpirationHandler:^{
        [application endBackgroundTask:backgroundTask];
        backgroundTask = UIBackgroundTaskInvalid;
    }];
}

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // Override point for customization after application launch.
    
    [self startServer];
    
    return YES;
}
{% endhighlight %}

safari中打开关键代码

{% highlight ruby linenos %}
__weak AppDelegate *appDelegate = (AppDelegate *)[[UIApplication sharedApplication] delegate];
    UInt16 port = appDelegate.httpServer.port;
    NSLog(@"%u", port);
    if (success) [[UIApplication sharedApplication] openURL:[NSURL URLWithString:[NSString stringWithFormat:@"http://localhost:%u/profile.mobileconfig", port]]];
    else NSLog(@"Error generating profile");
{% endhighlight %}
    
## 通过safari自带功能生成webclip
safari带有一个为当前网页生成webclip的功能，现在我们就需要使用这个方式来生成webclip。

这种方式的工作流程是这样的：先使用app打开safari并显示指定的网页内容（一般是指导用户怎么使用safari生成webclip,并打开safari的js控制），然后用户打开js权限，保存webclip,下次用户点击桌面上的webclip后即可再次打开该网页，触发js,跳回app。

因为用户点开webclip的时候需要获取到网页的所有信息，又因为我们的应用不是长时间在后台运行的，所以需要把所有网页的内容以url的形式中保存在webclip中,这种技术叫做data-url技术。

我们需要app把网页内容通过data-url的形式传给safari,我尝试过使用openurl的传输方式，这种方式不能传输太长的数据，行不通。所以这里的思路也是把app变成一个服务器，safari访问的时候返回302让TA重定向，重定向的url返回我们要传输的data-url,safari重定向后即可显示我们指定的网页内容。这里我们可以用基于[CocoaHTTPServer](https://github.com/robbiehanson/CocoaHTTPServer)之上封装的库[RoutingHTTPServer](https://github.com/mattstevens/RoutingHTTPServer)。

- 配置并传输data-url

{% highlight ruby linenos %}
    //配置返回值
    [appDelegate.httpServer get:@"/old" withBlock:^(RouteRequest *request, RouteResponse *response) {
        [response setStatusCode:302]; // or 301
        [response setHeader:@"Location" value:@"data:text/html;charset=UTF-8,<html><head><meta name='viewport' content='width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no'/><meta name='apple-mobile-web-app-capable' content='yes'/></head><body><img src=\"data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAJYAAACWCAYAAAA8AXHiAAAAGXRFWHRTb2Z0d2FyZQBBZG9iZSBJbWFnZVJlYWR5ccllPAAAAyNpVFh0WE1MOmNvbS5hZG9iZS54bXAAAAAAADw/eHBhY2tldCBiZWdpbj0i77u/IiBpZD0iVzVNME1wQ2VoaUh6cmVTek5UY3prYzlkIj8+IDx4OnhtcG1ldGEgeG1sbnM6eD0iYWRvYmU6bnM6bWV0YS8iIHg6eG1wdGs9IkFkb2JlIFhNUCBDb3JlIDUuNS1jMDE0IDc5LjE1MTQ4MSwgMjAxMy8wMy8xMy0xMjowOToxNSAgICAgICAgIj4gPHJkZjpSREYgeG1sbnM6cmRmPSJodHRwOi8vd3d3LnczLm9yZy8xOTk5LzAyLzIyLXJkZi1zeW50YXgtbnMjIj4gPHJkZjpEZXNjcmlwdGlvbiByZGY6YWJvdXQ9IiIgeG1sbnM6eG1wPSJodHRwOi8vbnMuYWRvYmUuY29tL3hhcC8xLjAvIiB4bWxuczp4bXBNTT0iaHR0cDovL25zLmFkb2JlLmNvbS94YXAvMS4wL21tLyIgeG1sbnM6c3RSZWY9Imh0dHA6Ly9ucy5hZG9iZS5jb20veGFwLzEuMC9zVHlwZS9SZXNvdXJjZVJlZiMiIHhtcDpDcmVhdG9yVG9vbD0iQWRvYmUgUGhvdG9zaG9wIENDIChNYWNpbnRvc2gpIiB4bXBNTTpJbnN0YW5jZUlEPSJ4bXAuaWlkOjIwQ0NGQkZERjlEMjExRTNBRjI0Rjg0NkE1MjQ1NkIyIiB4bXBNTTpEb2N1bWVudElEPSJ4bXAuZGlkOjIwQ0NGQkZFRjlEMjExRTNBRjI0Rjg0NkE1MjQ1NkIyIj4gPHhtcE1NOkRlcml2ZWRGcm9tIHN0UmVmOmluc3RhbmNlSUQ9InhtcC5paWQ6MjBDQ0ZCRkJGOUQyMTFFM0FGMjRGODQ2QTUyNDU2QjIiIHN0UmVmOmRvY3VtZW50SUQ9InhtcC5kaWQ6MjBDQ0ZCRkNGOUQyMTFFM0FGMjRGODQ2QTUyNDU2QjIiLz4gPC9yZGY6RGVzY3JpcHRpb24+IDwvcmRmOlJERj4gPC94OnhtcG1ldGE+IDw/eHBhY2tldCBlbmQ9InIiPz7zvoAyAAACkklEQVR42uzcvWoUYRSA4dkQFIyQ9F5AbDSCCcReBbFMJ7Ziu9PuBdhOUgbb4BUkCGqvYmPQwmBtH2JlmvUMOxZr9ptvskFQ53ngLGQmZwnDy/6RZDAeDsdF2nJRVSczz5Tl1bj9XrSz39P9hcwd3205d7/Is9/T/UHmEetLzJ2o9vi3Wlfi9m3M9cwPZr+n+7mwakcxo5jXzdf3Yp7FrBbd2O/hfpew4NwWXAKEhbAQFggLYSEsEBbCQlggLISFsEBYCAthgbAQFsICYfEXh7UesxRT/wHiRsxOzKlLQ0enTTNTHdV/pTPrm2/F7Mdcc91o8S3mYcxh16fCj82CRy5SfsQ8mBVV7jVWvfDc9SNhN+bTvC/e91w/El5c5F3hoetHwue2k4sXuuuqmv66LB3/n47n3w0mpd4V/rJZTP6rCMxq4/28T4WPXT8SHs37Gmst5onrR8LTmBvnDav+gPQg5pLrR8LlmJcxN1Nh3Y65Ukw+jq8/lt9unjt96k5O3ciHppmpjvzjNf4Iv92AsBAWwgJhISyEBcJCWAgLhIWwEBYIC2EhLBAWwkJYICyEhbBAWAgLYYGwEBbCAmEhLIQFwkJYCAuEhbAQFggLYSEsEBbCQlggLISFsEBYCAthgbAQFsICYSEshAXCQlgIC4SFsBAWCAthISwQFsJCWCAshIWwQFgIC2GBsBAWwgJhISyEBcJCWAgLhIWwEBYIC2EhLIQFwkJYCAuEhbAQFggLYSEsEBbCQlggLISFsEBYCAthgbD4N8P6GrMVs9zMVnOsK/s93R+Mh8Nx4txRzGZRVcdTR8tyJW7fxaxm7tt+j/fbHrFGZ+60Njk26hCt/R7vt4X1puXcqw4/mP0e7/8UYACYT0iYFm82ogAAAABJRU5ErkJggg==\" alt=\"\"></body><script>if (window.navigator.standalone){window.location.href='sample://';}</script></html>"];
    }];

//跳转
    UInt16 port = appDelegate.httpServer.port;
    NSLog(@"%u", port);
    [[UIApplication sharedApplication] openURL:[NSURL URLWithString:[NSString stringWithFormat:@"http://localhost:%u/old", port]]];

{% endhighlight %}

- 用户打开js

![_config.yml](/images/QQ20150309-2@2x.png)

- 通过safari保存webclip

![_config.yml](/images/QQ20150309-1@2x.png)

- data-url中加入js
	
    通过safari打开的html是处于safari mode,而直接通过webclip打开的html是处于app mode,可以理解为safari mode是嵌入在safari中的网页，app mode的网页是单独的网页，通过这个状态我们可以控制什么时候调用js,来控制最终是展示当前网页还是跳转到我们指定的app。这里我写的是 sample:// ,可以按照需要替换成app的scheme,即可跳转到app。

{% highlight js linenos %}
<script>if (window.navigator.standalone){window.location.href='sample://';}</script>
{% endhighlight %}

## 其他可以做的细节
- html和配置文件，我们都可以通过替换字符串等方式修改最终生成的内容，以此来针对不同用户生成不同内容。

{% highlight ruby linenos %}
NSString *templatePath = [[NSBundle mainBundle] pathForResource:@"phone_template" ofType:@"mobileconfig"];
   
    NSString *data = [NSString stringWithContentsOfFile:templatePath encoding:NSUTF8StringEncoding error:NULL];
   
    data = [data stringByReplacingOccurrencesOfString:@"!!NAME!!" withString:name];
    data = [data stringByReplacingOccurrencesOfString:@"!!PHONENUMBER!!" withString:phoneNumber];
   
    NSLog(@"%@", data);
   
    BOOL success = [data writeToFile:[ProfileGenerator profilePath] atomically:YES encoding:NSUTF8StringEncoding error:nil];
   
    return success;
{% endhighlight %}

- 端口号不一定要写死，这里仅仅是方便测试。

## 总结
2种方式都可以达到最终效果，选取哪种方式去实现，可以自己评估优劣。由于本人对服务端和前端不太熟悉，实现还有2点不足之处，希望有人能给出些比较好的方案。

1. 可能由于浏览器缓存的问题，如果之前safari打开过localhost:端口号，下次再进入时可能不会去重定向，导致webclip保存的不是重定向后的url,而是原本请求的url。
2. 重定向返回的response header长度这里也是有限制的，过长会造成截断，这里应该是可以通过代码改进的。
