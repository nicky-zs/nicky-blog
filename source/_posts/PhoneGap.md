---
title: iOS上PhoneGap与平台互操作
date: 2014-02-09 20:53:09
tags:
  - iOS
  - PhoneGap
---

PhoneGap是手机平台上流行的一款中间件。它构建在各种手机平台所提供的WebView（浏览器内核）组件的基础之上，使用javascript语言对应用开发者提供统一的接口（如调用相机、调用其他本地组件），从而屏蔽了各手机平台上OS的异构。在无线小组的调研任务中，我的任务主要是负责iOS平台上的调研，本文简单描述了iOS平台上PhoneGap与平台本地的互操作性的实现。

<!-- more -->

PhoneGap因为被捐赠给了Apache而改名为Cordova，所以PhoneGap里的类名都以CDV作为前缀。在iOS平台上，最重要的一个核心类是`CDVViewController`。该类直接继承自`UIViewController`，因而具备了所有`UIViewController`所具备的特性。同时，该类还实现了两个Protocol（即接口）：`UIWebViewDelegate`和`CDVCommandDelegate`。因此它也负责UIWebView的一些callback，以及`CDVInvokedUrlCommand`的执行。

`CDVViewController`类的主要对象成员是`CDVCordovaView *webView`，在源代码中可以看出，这个webView对象是`CDVViewController`的self.view上的唯一被add进来的子对象，即负责了整个`CDVViewController`类的显示。而`CDVCordovaView`类则没什么特别的，它直接继承自`UIWebView`。

当`CDVViewController`在构建时，它有两个很重要的属性：`NSString*wwwFolderName`和`NSString *startPage`。这两个属性值使得`CDVViewController`在load之后直接加载wwwFolderName下的startPage作为初始显示的页面。

以上是对`CDVViewController`的一个简单介绍。容易明白的是，在iOS应用中使用了`CDVViewController`之后，界面实际上就完全交给了`CDVCordovaView *webView`，即一个`UIWebView`。下面讨论互操作性问题。

首先，最简单的还是从iOS的进程中去操作webView中的DOM对象。因为从本质上说，webView的DOM对象肯定存在于当前进程的地址空间之中，因此从进程中去操作webView总会有比较方便的解决方案。实际上，UIKit框架本身已给我们提供了这样一个操作接口：`UIWebView`的`stringByEvaluatingJavaScriptFromString:(NSString *script)`方法。在webView上调用该方法，并传递一个字符串形式的javascript代码段作为参数，那么webView便会直接在其javascript运行时环境中执行这段javascript代码，并将结果以字符串的形式返回到进程的地址空间中。对于其他形式的数据对象而言，它们都可以使用JSON等格式序列化成一个字符串，从而用于参数的传递或者返回值的返回。所以理论上，这个接口可以完全满足从iOS上的进程中去操作webView中的DOM对象。

然而，困难一点的是在webView的运行时环境中去操作iOS进程中的对象。因为webView的javascript引擎的运行环境本身是对进程的地址空间毫无概念的。因而，要从webView的运行时环境中去操作iOS进程中的对象，一定需要进程自身的辅助。就目前的调研情况来看，大致有如下两种做法。

一、在进程中先向webView注册一些javascript对象，将这些javascript对象与本地对象进行绑定。然后在webView的javascript运行环境中便能直接通过注册的javascript对象来操作本地对象。Android平台、Mac OS平台都是这样做的，但是iOS平台不支持这种做法。

二、由于webView每次在发起请求之前，都会调用进程中的一个回调函数：`webView: shouldStartLoadWithRequest: navigationType:`，因此，重写该回调函数，在函数中获取webView发起的请求的url，然后解析该url，并根据具体的url来调用其他对象的函数。这是iOS平台支持的唯一办法。

通过看源代码，发现iOS平台上的PhoneGap便是利用这一做法。它会对webView所发起的每一次请求的url进行分析。如果发现url以 gap:// 开头，那么它不会执行本次请求，而是对webView执行一个javascript调用`cordova.require(‘cordova/plugin/ios/nativecomm’)()`从而得到一个Command队列，并且在本地依次对这个队列中的每一个Command进行执行。这里的Command是使用命令模式封装的对象名、方法名、参数列表等。程序会通过selector来完成对这个一个Command的调用。这样便达到了webView里的javascript操作到进程中对象的目的。

当然，如果弃用PhoneGap的这套模式，我们也完全可以自行解析url，然后在本地进行调用。

