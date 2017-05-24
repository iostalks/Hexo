---
title: UIKit 性能优化
date: 2017-04-24 21:49:25
tags: UI
---

UI（User Interface）是指手机屏幕上的图形界面，它由一个个的像素点构成，每个像素均由红、绿、蓝，三个独立的颜色单位组合，有时还带有透明度。屏幕每秒钟会进行 60 次绘制，在屏幕滚动时，如果 16.67ms(1/60) 内处理器无法提供完成刷新所需的数据，就会造成界面的卡顿。想要优化 UI 上的体验，必须对背后的原理一定了解，本文将介绍会影响屏幕流畅度的几个要点。

## 图层混合
混合图层，是指含有不透明度的图层相互叠加、计算后显示的效果，同不透明的图层相比，该计算需要额外的开销。两个像素点的叠加的计算方式，可以参考[这里](https://objccn.io/issue-3-1/)的合成一节。图层是树形结构的，如果只想显示最上层的视图，那就将其透明度设置为 100%，这样 GPU 就会忽略下面所有的图层，从而减少不必要的计算。

关于透明度 UIView 涉及到三个属性：alpha、isHidden、isOpaque。alpha 如果不为 100%，就会发生图层混合；isHidden 如果设置为 true 图层就不会被绘制，它通常等同于 alpha 设置为 0；isOpaque 属性默认为 true，这个属性有些特殊，表面上看它跟图层的透明度有关，然而实际上将其修改为 false，并不会影响图层的透明度。将其设置为 true 的作用是向绘制系统 "许诺" 即将绘制的每一个像素都要使用全不透明的颜色，绘制系统会忽略其下面的视图，从而避免图层混合。如果某个图层含有透明度，却将其 isOpaque 属性设置为 true，将发生不可预测的后果。然而我们实际操作中常常只是设置 alpha 的值，而没有将 isOpaque 设置为 false...

对于 UIImageView 来说，它不仅需要 UIImageView 本身不透明，它还不能含有 alpha 通道。我们在提交 AppStore 时，需要上传 1024 * 1024 的 icon，这个 icon 就不能含有 alpha 通道。macOS 系统自带的「预览」应用在文件导出 PNG 格式时，可以选择关闭 alpha 通道。

如下是不含有 alpha 通道，和含有 alpha 通道的对比
![image](http://oox2n98ab.bkt.clouddn.com/blend-comparison.png)

使用模拟器的 Color Blended Layers 查看，发生图层混合的部分呈现为红色。

## 离屏渲染
顾名思义，离屏渲染是指在屏幕之外作渲染，一般情况下，GPU 只渲染当前显示的内容，而离屏渲染会在屏幕外合成/渲染图层树的一部分到一个新的缓存区，然后等待该缓存区被渲染到屏幕上。

离屏渲染的合成计算是非常昂贵的，但并不一定是不好的。对于计算复杂的图层，如果复用率高（比如平移动画），使用离屏渲染的缓存，可以避免重复的计算。而对于重复利用率很低的图层，触发离屏渲染就得不偿失。

需要特别注意的是，单独设置 cornerRadius 并不会触发离屏渲染，只是视图的背景部分被设置成了透明。

会造成离屏渲染的几种情况：
- 设置 cornerRadius 圆角，并设置 maskToBounds/clipsToBounds 为 YES。
- 给 CALayer 设置 mask 蒙版
- 给 CALayer 设置阴影
- 设置光栅化

可使用模拟器的 Color Off-Screen Render 选项来查看触发离屏渲染的图层，显示颜色为黄色

## 光栅化
光栅化是指将一个图层 Layer 预先渲染成位图，然后加入到缓存，比如将含有阴影效果的图层进行缓存，可以一定幅度的提升性能。当 Layer 光栅化后会渲染成位图放入缓存，屏幕滚动的时，直接从缓存中读取。使用 Instrument 的 「Color Hits Green and Misses Red」来进行观察，命中缓存的 Layer 会呈现出绿色，未命中呈现红色。

对于性能要求不是特别高的，设置圆角最便捷的方式是 cornerRadius 配合 masksToBounds 来实现，而前面说到 masksToBounds 会导致离屏渲染。如果采用这种方式设置圆角，通过设置 shouldRasterize = true，使用光栅化将圆角缓存起来。


## 控件优化
#### UILabel
UILabel 的默认背景是透明的，而透明的视图为发生图层混合，所以将 UILabel 的背景色设置为父视图的颜色，虽然表面看上去没什么区别，实际避免了图层混合的开销。

UILabel 显示中英文的时候存在差别，含有中文的 Label，默认会创建一个透明 Content 的 Sublayer，同样会造成图层混合，解决方式是设置 label.layer.masksToBounds 为 true。

#### UIImageView
UIImageView 除了自身的要确保自身的 alpha 为 100%，还要尽可能的保证设置给 UIImageView 的图片 alpha 通道是关闭的。

保持 UIImageView 的 Frame 和图片的尺寸一直（Retain 屏下宽高比 2:1）。

大量圆角的 UIImageView 也是一大性能杀手，最优的方式是，直接将图片裁剪成圆角的。

## 总结
由于接触面有限，本文只是简单介绍了下优化的皮毛，更详细的内容可参考官方资料

[Quartz 2D Programming Guide](https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/drawingwithquartz2d/Introduction/Introduction.html)
[Building Concurrent User Interfaces on iOS](https://developer.apple.com/videos/play/wwdc2012/211/)
[Optimizing 2D Graphics and Animation Performance](https://developer.apple.com/videos/play/wwdc2012/506/)
[Advanced Graphics and Animations for iOS Apps](https://developer.apple.com/videos/play/wwdc2014/419/)
[Practical Drawing for iOS Developers](https://developer.apple.com/videos/play/wwdc2011/129/)

## 参考
[iOS 保持界面的流畅技巧](http://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)
[UIKit性能调优实战讲解](http://www.jianshu.com/p/619cf14640f3)



