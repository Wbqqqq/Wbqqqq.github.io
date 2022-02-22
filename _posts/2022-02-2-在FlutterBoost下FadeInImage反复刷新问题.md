---
layout: post
title: "在FlutterBoost下FadeInImage反复刷新问题"
date: 2022-02-22 14:00
---

### 问题背景

路由框架采用flutterboost，根视图UITabbarController的其中一个item的页面是flutter，该flutter页面采用了FadeInImage显示图片，其余都是native。在剩余的native中可以点击进入flutter页面。直接看现象:

![2021-09-15 14.41.42.gif](https://upload-images.jianshu.io/upload_images/2782305-2c86feedd74e152a.gif?imageMogr2/auto-orient/strip)








结果:只要切换了别的flutter页面再回内嵌在TabbarItem的flutter页面，页面中的图片就会重新动画。



开始以为是自己的项目哪里没写对，想着放到flutterboost的example上尝试一下，

在tabItem页随便添加一个widget:

```dart
FadeInImage.assetNetwork(
  placeholder: "images/flutter.png",
  image: "https://gw.alicdn.com/tfs/TB1aUlEYLb2gK0jSZK9XXaEgFXa-252-252.png",
  width: 300,
  height: 300,
  fadeOutDuration: const Duration(milliseconds: 300),
  fadeInDuration: const Duration(milliseconds: 700),
),
```

现象如下:

![2021-09-16 11.01.43.gif](https://upload-images.jianshu.io/upload_images/2782305-473c557b116698ad.gif?imageMogr2/auto-orient/strip)


结果:同样的场景，得到了一个更让人奇怪的现象，切换了tab就会重新动画，不切tab怎么玩都没关系。





### 排查过程

1.首先先查明为什么会刷新图片，来到`FadeinImage`内部的build方法

![image-20210915154239430.png](https://upload-images.jianshu.io/upload_images/2782305-21526ed52e7a1503.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



通过断点调试最终发现图片是通过`image`中`framBuilder`回调创建视图，而该回调中的`wasSynchronouslyLoaded`决定了是返回网络图片，还是返回占位图动画。所以再跟进去看一下。

```
官方注解:如果framBuilder为null，一旦第一个图像帧可用，则此小部件将显示绘制为的图像。调用方也可以使用此生成器向图像添加效果（例如在图像变暗时淡入）或在加载图像时显示占位符小部件。
```

2.看一下Image的build方法，因为image是一个statefulwidget，所以这里是imageState

![image-20210915160330391.png](https://upload-images.jianshu.io/upload_images/2782305-c6d92ad1a1a3e086.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


也验证了刚才的说的官方注解。那么就看一下为什么tab切换这里返回的是`wasSynchronouslyLoaded`是`fasle`，其他情况都是`true`。

3.查看Widget树，发现两种切换方式层级都是一样的。

![image-20210915165550149.png](https://upload-images.jianshu.io/upload_images/2782305-69481a9e7bbf75a8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


众所周知，当同级的子树发生变化的时候，都用`didChangeDependencies()`，每次都会进入`_updateSourceStream`

![image-20210915172035541.png](https://upload-images.jianshu.io/upload_images/2782305-9961a35432a4d4ba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![image-20210915172256536.png](https://upload-images.jianshu.io/upload_images/2782305-eea5c04b9f3bb5b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


关键可以看到这个方法，关键在于如果`_imageStream.key`不相等，wasSynchronouslyLoaded就设置成false了。如果正常加载过一次，该值默认就是true，这个这里就不展开了。这里的`key`是stream的completer对象。

![image-20210915173815574.png](https://upload-images.jianshu.io/upload_images/2782305-d65d0dd66cbcab4b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


4.到这就更奇怪了，为什么URL都是一样的，`_imageStream`为啥会不一样呢？这里主要就要看下这个newStream是怎么拼出来的了。这块代码会有点深，说结论：

![image-20210915174053453.png](https://upload-images.jianshu.io/upload_images/2782305-9c18723b39e6e7b4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![image-20210915173928584.png](https://upload-images.jianshu.io/upload_images/2782305-336c139be5a65e32.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


imageCache的_cache属性如果有key，直接返回image.completer。所以通过断点发现，切换tab。

##### 到这里的结论就是这里的_cache在某个时刻清空了，导致返回的`key`（image.completer）不一致了。

5.看看_cache在何时清空的:

![image-20210915180522047.png](https://upload-images.jianshu.io/upload_images/2782305-095ca83692169db5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


看起来是从native调过来的，

`channel`和`type`分别是`flutter/system`和`memoryPressure`

![image-20210916103914716.png](https://upload-images.jianshu.io/upload_images/2782305-725e7f02710494c0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![image-20210916103843272.png](https://upload-images.jianshu.io/upload_images/2782305-0e2a720e1a0b9a8e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


6.调试一下engine看看怎么传过来的。

全局搜一下engine源码，看起来是这个。

![image-20210916104304137.png](https://upload-images.jianshu.io/upload_images/2782305-8965ebba825726fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


lldb打上方法断点看看什么时候调:

![image-20210916105009888.png](https://upload-images.jianshu.io/upload_images/2782305-ed871f80844abcfb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


快知道最终原因了:

![image-20210916105109496.png](https://upload-images.jianshu.io/upload_images/2782305-9c4610937f677871.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image-20210916105141640.png](https://upload-images.jianshu.io/upload_images/2782305-bf6e8e5678d194e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


因为flutter_boost把engine.viewController设置为了nil，触发了`memoryPressure`。

方法的源头是`detachFlutterEngineIfNeeded`

而`detachFlutterEngineIfNeeded`就是在vc dismiss的时候调用。

![image-20210916104755214.png](https://upload-images.jianshu.io/upload_images/2782305-e5a3e6ef3a442e55.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


7.到这更奇怪了，两种方式都是dismiss，为什么切换tab，会重新动画呢？

![image-20210916105535275.png](https://upload-images.jianshu.io/upload_images/2782305-136df67c6dec1c1f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 结论
最终答案在这，如果是当前页面push和pop会进入`attatchFlutterEngine`的逻辑，而在简单场景下例如A->B->A detachFlutterEngine方法内部因为判断self.viewController.engine != self，所以就不会执行之后的逻辑。而嵌tabbar页中flutter vc在flutterboost看起来，也做了相同的逻辑，一旦切出，我也默认这个页面销毁了，不再进行管理。切换tab，FadeInImage重新做动画的根本原因就是，FlutterBoost在`detachFlutterEngine`把engine.viewcontroller置为了nil，触发了`memoryPressure`(内存压力)，把ImageCache清空了，导致虽然网络图片的缓存还在，但是让flutter误以为需要重新加载。


稍微改一下代码

![image-20210916110126681.png](https://upload-images.jianshu.io/upload_images/2782305-5b4aea39f193256b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![2021-09-16 11.01.43.gif](https://upload-images.jianshu.io/upload_images/2782305-68217ed22330c06a.gif?imageMogr2/auto-orient/strip)


完美。



暂时不知道官方这么写的原因，所以最后的解决方法，还是把动画时间调成最短，让人肉眼看不到图片重新做动画。



补充核心过程(假设A是tab-native，B是tab-flutter，B的图片已经加载完毕):

第一种情况(B->A->C->A->B)

第一步:回到A，push一个flutter的C页面，触发`memoryPressure`（init），并且因为层级变化image触发`didChangeDependencies`

![image-20210916144339234.png](https://upload-images.jianshu.io/upload_images/2782305-ea1ce1798c686e0a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


可以看到`_cache`清空了，但`_liveImages`中的网络图片对象还在，这个时候会把_liveImages的对象取出，重新放到`_cache`中。
![image-20210916144742323.png](https://upload-images.jianshu.io/upload_images/2782305-d1a2f851aeab49e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


并且返回对象。这个时候`_cache`又恢复正常。

![image-20210916144828593.png](https://upload-images.jianshu.io/upload_images/2782305-8c88b6c0b6827eff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


之后因为image不需要在响应动画，所以移除通知

![image-20210916153426636.png](https://upload-images.jianshu.io/upload_images/2782305-5bf4168f331c42d4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


再移除的过程中将`liveImage`清掉

![image-20210916153507713.png](https://upload-images.jianshu.io/upload_images/2782305-9fbb91af4c625dfc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


所以进入C页面的最后结果是

![image-20210916153552069.png](https://upload-images.jianshu.io/upload_images/2782305-275bee8221d78a76.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


第二步.C dismiss，触发`memoryPressure`（engineDetach），清了一把`_cache`

![image-20210916153714071.png](https://upload-images.jianshu.io/upload_images/2782305-5c1a1bdd316d2d20.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这个时候什么缓存都没了，Image的key需要重新创建初始化，所以回到B会重新动画。


第二种情况:（B->C->B）

第一步同上。

第二步.C dismiss，不会`engineDetach`，所以不会触发`memoryPressure`，所以回到B，直接拿缓存，不用重新创建。




### 后续
给官方提了一个PR，新增vc keepalive属性，针对这种tab内嵌的页面进行特殊管理。
https://github.com/alibaba/flutter_boost/pull/1422?w=1
































