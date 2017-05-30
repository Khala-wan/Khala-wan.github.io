---
layout: post
title:  "iOS-骚操作-UIDebuggingInformationOverlay"
date:   2017-05-27 22:48:45
description: UIDebuggingInformationOverlay是一个新的工具帮助我们更好地调试我们的APP.
categories:
- blog
- iOS
---


UIDebuggingInformationOverlay是一个私有的UIWindow的子类。它的作用就是用来帮助iOS的开发人员和设计人员用来调试自己的APP。值得一提的是这个功能(Window)是个私有类。是前几天才被Ryan Peterson在浏览UIKit的私有头文件的时候发现并公布给大家。这是他的英文原文博客[UIDebuggingInformationOverlay](http://ryanipete.com/blog/ios/swift/objective-c/uidebugginginformationoverlay/).

## UIDebuggingInformationOverlay能帮我们干什么
正如之前所说，UIDebuggingInformationOverlay能帮我们在APP运行时调试UI的各种问题。
大致如下：
* View Hierarchy 
* VC Hierarchy
* Ivar Explorer
* Measure
* Spec Compare
* System Color Audit（暂时没发现有什么用）

![img](https://github.com/Khala-wan/Khala-wan.github.io/raw/master/resource/UIDebuggingInformationOverlay-1.jpg)

## View Hierarchy (查看视图层级)
View Hierarchy可以帮我们查看当前控制器视图的层级关系，之前这些工作是Reval和Xcode的Debug View Hierarchy（断点调试）来帮我们完成的。Reval确实很好用，但它毕竟是一个第三方应用。相对来说，我更愿意使用Apple自己的工具。（摊手）
如图：

点击Change Current Window可以切换当前Debug的主Window（如果你有多个window的话）。
下面的图以某个UIImageView为例：
点击单个View层右侧的按钮可以详细查看这个View的类型、不透明度（可调节）、属性、状态等等。一般我们比较关注的是某个View的Frame。

查看View的属性可以找到这个类对应的属性值和我们所关心的数值进行比对。

查看状态信息可以看到这个View的类型和一些纯视图的属性如Transform等。不过每种类型的View其State Info中的Value的类型、数量也不同。

## VC Hierarchy (查看控制器层级)
这个功能可以帮助你定位当前Winow的控制器关系，同时他也可以看出你Present之后的控制器层级关系。

同样的，点击某一个你想要关注的VC就可以看到它对应的一些信息。也可以继续点击它的UIView来查看这个控制器所对应UIView的信息。

## Ivar Explorer （变量浏览）
这个功能有点吊，它可以让你浏览UIApplication的各种变量，更重要的是，还可以探索其他对象的变量。 
比如我在UIApplicationDelegate中定义了一个变量，用来判断APP开启后是否需要跳转到消息中心。
``` swift
//是否跳转消息中心
var needToMessage:Bool = false
```
然后我们现在来想办法找到它，看看能不能看到它的值。首先进入Ivar Explorer。
然后找到UIApplication的delegate对象。点击去看一下，第二个就是我们要找的变量。当前它的值是NO，看来我们不需要跳转到消息中心了。至于你还想在你的APP中看到哪个对象的哪个变量就看你自己的咯~

## Measure (测量工具)
呃。。。讲真的这个功能绝对是我们和UI撕逼的双刃剑。有了它，你不用再因为那么0.5px的问题争论半天。但是如果UI会用它了，那么你可能还要多几个0.5px的问题😂
点击Vertical并长按你想测量的地方就可以显示出垂直方向上对应的间距，还可以按住不动拖动手指测量别的地方。如果想测量水平方向，你懂得，点Horizontal。细心的你会发现还有个叫View Mode的Switch。如果你打开了它，你的测量就会基于每一个View进行测量，效果如下：

## Spec Compare (UI对比工具)
这个功能可以让我们在对比UI图和实际开发出来的界面时不用两个页面看来看去。只需要从相册选择你这个页面的UI截图。然后向下滑动就可以增加透明度以比对细节。效果如下(UI图我找不到了。先用系统的凑合下)：


>⚠️注意因为要弹出UIImagePickerController所以在iOS9以上别忘了在Info.plist中加入相册使用的key:NSPhotoLibraryUsageDescription。否侧会闪退。

## 使用UIDebuggingInformationOverlay
说了那么多，那么要怎么使用它呢。
OC：
```objectivec
Class overlayClass = NSClassFromString(@"UIDebuggingInformationOverlay");
[overlayClass performSelector:NSSelectorFromString(@"prepareDebuggingOverlay")];
id obj = [overlayClass performSelector:NSSelectorFromString(@"overlay")];
//呼出UIDebuggingInformationOverlay窗口
[obj performSelector:NSSelectorFromString(@"toggleVisibility")];
```
Swift:
``` swift
let overlayClass = NSClassFromString("UIDebuggingInformationOverlay") as? UIWindow.Type
_ = overlayClass?.perform(NSSelectorFromString("prepareDebuggingOverlay"))
let overlay = overlayClass?.perform(NSSelectorFromString("overlay")).takeUnretainedValue() as? UIWindow
//呼出UIDebuggingInformationOverlay窗口      
_ = overlay?.perform(NSSelectorFromString("toggleVisibility"))
```
在你想呼出UIDebuggingInformationOverlay窗口的地方只需要调用toggleVisibility()方法即可。或者更方便的方法是双击两下status bar。

## 总结
UIDebuggingInformationOverlay从各个方面来看都是很有用的，我也不知道Apple为什么不把它公布出来。但是至少现在我们还是可以用它的。**记得在提交AppStore的时候把它去掉，不然会被拒的。**
如果你还发现了其他酷炫的用法，请一定告诉我。3q~
