---
layout: post
title: 导航自定义动画以及手势返回和手势前进
---

##### 前言

最近整理框架的时候，研究了下ios7后苹果提供的自定义导航动画协议，下面内容分享给大家。先上[整理好的导航框架demo](https://github.com/pingyourid/ShuttleNavigationController)。

##### 效果图

<img src="/images/back_foward.gif" alt="" title="" width="375" />

##### 导航协议和导航的基础知识 

1

{% highlight ruby linenos %}
- (void)navigationController:(UINavigationController *)navigationController willShowViewController:(UIViewController *)viewController animated:(BOOL)animated;
{% endhighlight %}

push和pop方法被调用后，动画执行前会被调用。

2

{% highlight ruby linenos %}
- (void)navigationController:(UINavigationController *)navigationController didShowViewController:(UIViewController *)viewController animated:(BOOL)animated;
{% endhighlight %}

push和pop方法被调用后，动画执行结束会被调用，如果动画被取消，此处不会执行。

3

{% highlight ruby linenos %}
- (id<UIViewControllerInteractiveTransitioning>)navigationController:(UINavigationController *)navigationController interactionControllerForAnimationController:(id<UIViewControllerAnimatedTransitioning>)animationController
{% endhighlight %}

返回一个交互式内容，返回的内容内的当前手势完成百分比信息可以控制系统的动画完成度百分比。所谓交互式，就是手可以控制动画的意思。

4

{% highlight ruby linenos %}
- (id<UIViewControllerAnimatedTransitioning>)navigationController:(UINavigationController *)navigationController animationControllerForOperation:(UINavigationControllerOperation)operation fromViewController:(UIViewController *)fromVC toViewController:(UIViewController *)toVC
{% endhighlight %}

返回一个动画，实现这个协议后，系统自带的左滑返回动画会不起作用，如果这里动画返回nil,系统不会去调用流程3.

5

{% highlight ruby linenos %}
- (void)pushViewController:(UIViewController *)viewController animated:(BOOL)animated
{% endhighlight %}

系统的push方法。

6

{% highlight ruby linenos %}
- (UIViewController *)popViewControllerAnimated:(BOOL)animated
{% endhighlight %}

系统的pop方法。

执行流程是：

5/6，1，4，如果4返回不为nil,3，如果3不为nil或者3中使用手势让动画完成，2

5/6，1，4，如果4返回nil,2

5/6，1，4，如果4返回不为nil,3，如果3为nil,2

5/6，1，4，如果4返回不为nil,3，如果3不为nil并通过手势取消动画.


##### 交互式动画

也就是给导航加上手势，手势去触发动画，并可以通过手势控制动画。

7

{% highlight ruby linenos %}
- (void)prepareGestureRecognizer
{
    UIGestureRecognizer *gesture = self.interactivePopGestureRecognizer;
    gesture.enabled = NO;
    UIView *gestureView = gesture.view;

    //pop
    UIScreenEdgePanGestureRecognizer *popRecognizer = [[UIScreenEdgePanGestureRecognizer alloc] initWithTarget:self.customAnimationDelegate action:NSSelectorFromString(@"handleEdgePanGestureRecognizer:")];
    popRecognizer.delegate = self.customAnimationDelegate;
    popRecognizer.edges = UIRectEdgeLeft;
    objc_setAssociatedObject(popRecognizer, @"direction", @"pop", OBJC_ASSOCIATION_COPY);
    [gestureView addGestureRecognizer:popRecognizer];

    if (self.enableInteractivePush) {
        //push
        UIScreenEdgePanGestureRecognizer *pushRecognizer = [[UIScreenEdgePanGestureRecognizer alloc] initWithTarget:self.customAnimationDelegate action:NSSelectorFromString(@"handleEdgePanGestureRecognizer:")];
        pushRecognizer.delegate = self.customAnimationDelegate;
        pushRecognizer.edges = UIRectEdgeRight;
        objc_setAssociatedObject(pushRecognizer, @"direction", @"push", OBJC_ASSOCIATION_COPY);
        [gestureView addGestureRecognizer:pushRecognizer];
    }
}
{% endhighlight %}

给导航添加返回手势和前进手势。

8

{% highlight ruby linenos %}
- (void)handleEdgePanGestureRecognizer:(UIScreenEdgePanGestureRecognizer *)recognizer
{
BOOL isPush = NO;
if ([objc_getAssociatedObject(recognizer, @"direction") isEqualToString:@"push"]) {
    isPush = YES;
}

CGFloat progress = [recognizer translationInView:recognizer.view].x / recognizer.view.bounds.size.width;
if (isPush) { //push的时候最右边是0,所以会是负数
    progress = -progress;
}

/**
 *  稳定进度区间，让它在0.0（未完成）～1.0（已完成）之间
 */
progress = MIN(1.0, MAX(0.0, progress));
if (recognizer.state == UIGestureRecognizerStateBegan) {
    self.interactiveTransition = [[UIPercentDrivenInteractiveTransition alloc] init];

    if (isPush) {
        [self.ownerNC pushViewController:self.ownerNC.vcHolder.lastVC animated:YES];
    }
    else {
        [self.ownerNC popViewControllerAnimated:YES];
    }
}
else if (recognizer.state == UIGestureRecognizerStateChanged) {
    [self.interactiveTransition updateInteractiveTransition:progress];
}
else if (recognizer.state == UIGestureRecognizerStateEnded || recognizer.state == UIGestureRecognizerStateCancelled) {
    /**
     *  手势结束时如果进度大于一半，那么就完成pop操作，否则重新来过。
     */
    if (progress > 0.5) {
        [self.interactiveTransition finishInteractiveTransition];
    }
    else {
        [self.interactiveTransition cancelInteractiveTransition];
    }

    self.interactiveTransition = nil;
}
}
{% endhighlight %}

手势被触发后，根据手势进度去调用push或者pop操作，并计算手势进度传给要返回给系统的交互式内容interactiveTransition.

如果从手势触发，流程会是这样：

8，5/6，......

5和6后的流程同上。

##### 导航动画的完成的回调

原先的想法是先将函数指针从外部传递进来，最后通过1和2来调用block实现，但是考虑到有些流程会导致2不被调用，所以将回调的函数指针传入自定义动画中，让自定义动画根据执行结果来调用函数。

{% highlight ruby linenos %}
completion:^(BOOL finished) {
          if ([transitionContext transitionWasCancelled]) {
              toView.layer.transform = CATransform3DIdentity;
              toView.alpha = 1.0;
          }
          [transitionContext completeTransition:![transitionContext transitionWasCancelled]];
          if (self.comleteBlock) {
              self.comleteBlock(!transitionContext.transitionWasCancelled);
          }
        }];
{% endhighlight %}

##### 导航前进功能

该功能需要建立一个栈，保存需要前进的对象，在push的时候丢掉保存的对象，在pop的时候存入对象。

- 入栈和出栈地点选择。

2中由于只有目标vc，没有from vc,所以不能完成这个任务。并且这个时候从navigation中去取，也取不到fromvc,因为这个方法在popViewController之后被执行，所以fromvc已经从系统导航栈了去除了。

5/6 中，由于5/6类似的函数比较多，要重写好几个，才能实现这个功能，所以放这里不算最好的。

8中，这个地方是手势触摸才会被调用，点击按钮的方式不会走这边，排除

最后把地点选在4中，这边任何导航操作都会走，并且能取到fromvc和tovc,还可以根据动画执行结果判断是否执行入栈出栈操作。

- 被push后viewcontroller生命周期

由于view不是重新创建的，所以会想pop一样，从viewWillAppear开始执行，这样很赞。

##### 补充

考虑到保存很多viewController后内存会不会有问题，所以可能会加上栈内容量最大值的约束。

##### 参考文章和参考的开源库

[iOS利用Runtime自定义控制器POP手势动画 - CocoaChina 苹果开发中文站 - 最热的iPhone开发社区 最热的苹果开发社区 最热的iPad开发社区](http://www.cocoachina.com/ios/20150401/11459.html)

https://github.com/ColinEberhardt/VCTransitionsLibrary

https://github.com/zys456465111/CustomPopAnimation