---
layout: post
title: Interface Orientation
date: 2016-06-27
tags: UI
---

通过了解屏幕旋转的基本过程，以及对比 iOS5 及以前与 iOS6 及以后屏幕旋转 API 的异同，理解与实现 iOS6 后支持每个 ViewController 控制自身屏幕方向旋转的需求。

### 屏幕旋转事件
1. 当 iOS 设备方向发生改变时，系统会发送 UIDeviceOrientationDidChangeNotification 通知，默认情况下 UIKit 框架会监听这个通知并使用它自动更新界面方向。
2. 当 UIKit 接收到方向通知时，它会使用 UIApplication 对象和根视图控制器决定新的方向是否允许。也就是 Application 和 UIViewController 设置支持的方向对比如果都支持新的方向，那么就会旋转成新的方向，否则就忽略设备方向的改变。另外如果对比结果没有任何一个方向支持，就会抛出 UIApplicationInvalidInterfaceOrientationException 异常。

### 屏幕旋转 API

#### iOS6 之后

#### supportedInterfaceOrientations 声明视图控制器支持的界面方向

重写这个方法的 UIViewController 必须符合下面两个条件之一才有效：

1. 这个 UIViewController 必须是 window 的 rootViewController。
2. 这个 UIViewController 是通过 Modal ( `presentViewController:` )方式展示出来的。

默认情况下，设备上的视图控制器使用 iPad 的约定支持全部四个方向。而使用 iPhone 约定的设备，除了上下倒转的方向其它都支持。

例子：

```objc
- (NSUInteger)supportedInterfaceOrientations
{
    return UIInterfaceOrientationMaskPortrait | UIInterfaceOrientationMaskLandscapeLeft;
}
```

#### shouldAutorotate 动态控制是否旋转

这个方法会在执行任意自动旋转之前被调用，重写这个方法用于控制是否支持旋转。和 supportedInterfaceOrientations 方法一样必须符合两个条件之一才有效。

#### preferredInterfaceOrientationForPresentation 声明首选呈现方向

这个方法返回 ViewController 初始化时的方向。首选界面方向必须是视图控制器所支持的方向之一。和 supportedInterfaceOrientations 方法一样必须符合两个条件之一才有效。


#### iOS5 之前

在 iOS5 与之前所有视图控制器都可以通过重写 `shouldAutorotateToInterfaceOrientation` 方法，根据传入即将要旋转的方向参数决定是否旋转。但 Apple 文档建议：

在实际操作时，子级覆盖父级的能力很少用到。考虑到这一点，你应该尽可能在必须支持 iOS 5 的应用程序中模拟 iOS 6 的行为：

- 在根视图控制器或全屏幕显示的视图控制器中，选取对你的用户界面有意义的一部分界面方向。
- 在子级控制器中，通过设计适应性强的视图布局支持所有默认分辨率。

例子：

```objc
- (BOOL)shouldAutorotateToInterfaceOrientation:(UIInterfaceOrientation)orientation
{
   if ((orientation == UIInterfaceOrientationPortrait) ||
       (orientation == UIInterfaceOrientationLandscapeLeft))
      return YES;
 
   return NO;
}
```

### Rotation Callbacks

旋转被触发时事件发生的顺序：

1. `willRotateToInterfaceOrientation:duration:` 开始旋转前调用，可以重写这个方法在你的自定义内容视图控制器中隐藏视图或界面被旋转之前对视图布局做其它更改。

2. 窗口调整视图控制器的视图的边界时，这会导致视图布局它的子视图，触发视图控制器的 `viewWillLayoutSubviews` 方法。

3. `willAnimateRotationToInterfaceOrientation:duration:` 这个方法会在旋转动画Block内部被调用，在这个方法内修改的属性会作为动画的一部分。

4. 动画效果被执行。

5. `didRotateFromInterfaceOrientation:` 在旋转结束后调用。

这些事件会从 ContainerViewController 中自动传递到所有的 ContentViewController 中也就是 childViewControllers (iOS8.3 测试中需要 ViewController 的 View 添加到 ContainerViewController 的 View  中才会调用)，如果想禁止这个自动传递的特性那么在 iOS6 之后重写 `shouldAutomaticallyForwardRotationMethods` 方法并返回 No 即可，但是这个方法在 iOS8 之后已经失效。取而代之的是 `viewWillTransitionToSize:withTransitionCoordinator:` 这个方法只要 ContainerViewController 或 childViewControllers 中有一个 ViewController 重写该方法上面 iOS8 之前的那三个方法就不会被调用，而会调用该方法，并且该方法默认会调用所有 childViewController 的这个方法实现。

Apple 官方文档在旋转时处理的一些提示：

> 根据你的视图的复杂度，你可能需要写大量的代码支持旋转－或者完全不写。当搞清楚你需要做什么时，你可以使用下面的提示作为书写代码时的指南。
>
>- **在旋转的期间临时禁止事件发出。**对视图禁用事件发出防止不必要的代码在处理旋转更改时执行。
>- **存储可见地图区域。**如果你的应用程序包含地图视图，在开始旋转之前保存可见地图区域值。当旋转完成后，根据需要使用已保存的值确保显示区域与之前大概相同。
>- **对于复杂的视图层次结构，替换视图为快照图像。**如果动画大量的视图会导致性能问题，可以临时将这些视图替换为包含这些视图图像的图像视图。旋转完成之后，重新安装视图并移除图像视图。
>- **旋转之后重新加载全部可见表格的内容。**当旋转完成后强制进行重新加载操作确保所有新暴露的表格行会被适当的填充。
>- **使用旋转通知更新你的应用程序的状态信息。**如果你的应用程序使用当前方向判断如何呈现内容，使用视图控制器的旋转方法 (或相应的设备方向通知) 记录这些更改并做必要的调整。

随着 Autolayout 的普遍使用 API 的更新 ( iOS8 替换的旋转回调`viewWillTransitionToSize:withTransitionCoordinator:`) 貌似 Apple 也删除了这份比较针对 Frame 方式处理的文档。

### 设置 Interface Orientation 

一、Info.plist 或 target-General 中设置。

二、AppDelegate 中实现 `application:supportedInterfaceOrientationsForWindow:` 方法返回支持的屏幕方向。

> 在以上两个地方设置支持的方向会影响整个应用所支持旋转的方向，也就是如果以上两个地方都不支持如`横屏向左`方向的旋转那么在该应用中其它地方设置支持该方向的旋转，也是无法旋转的！
> 
> 在实际 iOS8.3 与 iOS9.3 的测试中发现在 `application:supportedInterfaceOrientationsForWindow:` 方法中设置的方向优先于在 Info.plist 中设置的方向。也就是说在这两个地方中同时设置了支持的方向，只会参考 `application:supportedInterfaceOrientationsForWindow:` 方法的设置，而不理会在 Info.plist 中的设置。当然如果只在 Info.plist 设置，而未通过重写 `application:supportedInterfaceOrientationsForWindow:` 方法进行设置，那么会参考 Info.plist 中的设置。
> 
> 通过网上了解到也可能是在未重写 `application:supportedInterfaceOrientationsForWindow:` 方法时，系统会读取 Info.plist 中支持的方向参数并在 `application:supportedInterfaceOrientationsForWindow:` 方法中返回。所以这两个地方的设置可以看作是同一个设置(未验证)。当然貌似在提交 App Store 时会读取你应用的 Info.plist 来获取应用支持的屏幕方向，所以不管是否自己实现了 `application:supportedInterfaceOrientationsForWindow:` 方法最好还是在 Info.plist 中设置支持的所有方向。

三、在 ContainerViewController 中设置，也就是在 `UIViewController` 或其子类中如`UINavigationController`、`UITabBarController`、等当然也可以是 Custom Container View Controller 中， 通过重写方法。iOS6 以前是 `shouldAutorotateToInterfaceOrientation:` 根据传入即将要旋转的方向参数决定是否旋转。 iOS6之后是 `shouldAutorotate` 和 `supportedInterfaceOrientations` 两个方法决定是否旋转的。只有当 `shouldAutorotate` 方法返回 YES 才会根据 `supportedInterfaceOrientations` 方法返回支持旋转的方向决定当前 ViewController 是否支持该方向的旋转。如果 `shouldAutorotate` 方法返回 NO 那么就不管 `supportedInterfaceOrientations` 方法返回什么当前 ViewController 都不会旋转。

> 需要注意的是在 ContainerViewController 中设置的方式需要至少复合下面两个条件中的一个设置才有效，不然即使实现上述的两个方法也没有作用，而且这两个方法都不会被调用到。
>
> 1. ContainerViewController 必须是 window 的 rootViewController。
> 2. ContainerViewController 是通过 Modal ( `presentViewController:` )方式展示出来的。
>
> 另外通过这种方式的设置会作用于当前 ContainerViewController 和 childViewControllers 内的所有 ViewController ，例如是 `UINavigationController` 那么所有 push 进去的 ViewController 支持的旋转方向都由这个 NavigationController 决定。如果是 Modal 方式展示的 ContainerViewController 那么也一样。

### 初始化方向

在 `UIViewController` 中还有一个 `preferredInterfaceOrientationForPresentation` 方法，这个方法返回 ViewController 初始化时的方向。这也属于上面第三种的设置方式 ContainerViewController 中设置，所以也必须符合上面两个条件中的一个，这个设置才有效，方法才会被调用。并且如果是以 Modal 方式展示的那么可以设置在上面第一第二种方式( Info.plist 和 `application:supportedInterfaceOrientationsForWindow:` 方法)中不支持的方向,也能旋转。

### 强制旋转屏幕
这里只列举一个比较简单的。

```objc
// 有人测试过可以提交App Store
NSNumber *orientation = [NSNumber numberWithInteger:self.isFullScreen?UIInterfaceOrientationLandscapeLeft:UIInterfaceOrientationPortrait];
[[UIDevice currentDevice] setValue:orientation forKey:@"orientation"];
```

### iOS6 后支持每个 ViewController 控制自身屏幕方向

通过上面我们知道在 iOS6 后屏幕旋转相关的三个 API `shouldAutorotate` `supportedInterfaceOrientations` `preferredInterfaceOrientationForPresentation` 必须符合两个条件中的一个，也就是当前 viewController 是 rootViewController 或者是 Modal 模式展示时才有效。因此要在 iOS6 以后的 API 中实现 iOS5 时每个 ViewController 通过 `shouldAutorotateToInterfaceOrientation` 方法控制自身屏幕旋转方向的效果。我们只需要重写这三个方将他们代理给当前 ContainerViewController 中最顶层的 ViewController 的方法即可。

通过 Category 实现具体代码如下：

```objc
@implementation UINavigationController (JKOrientation)

- (BOOL)shouldAutorotate
{
    return self.topViewController.shouldAutorotate;
}

- (NSUInteger)supportedInterfaceOrientations
{
    return self.topViewController.supportedInterfaceOrientations;
}

- (UIInterfaceOrientation)preferredInterfaceOrientationForPresentation
{
    return self.topViewController.preferredInterfaceOrientationForPresentation;
}

@end
```

### 参考资料

1. [iOS 翻译 《View Controller Programming Guide for iOS：Supporting Multiple Interface Orientations》](http://humyang.github.io/2015/VCP8/)
2. [IOS Orientation](http://www.cnblogs.com/jhzhu/p/3480885.html)