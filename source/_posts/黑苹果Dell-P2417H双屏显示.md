---
title: 黑苹果Dell P2417H双屏显示
date: 2019-11-24 10:36:11
tags: 
- 黑苹果
categories: Hackintosh
---
台式机装了黑苹果，能够双屏显示，但是会出现字体模糊的情况。了解到苹果有HiDPI这种黑科技来提升显示效果，所以就试了一下，然后就开启了漫长的折腾之路。
有关Hi-DPI可以看一下这篇文章：[HiDPI是什么？以及黑苹果如何开启HiDPI](https://www.sqlsec.com/2018/09/hidpi.html)
<!-- more -->
# 配置
- 屏幕型号：Dell P2417H
- 分辨率：1920x1080
- 黑苹果系统版本：Mojave 10.14.5
- 显卡：Intel HD630、NVIDIA GT710
- 显示器接口：hdmi

# 开启HiDPI
网上开启HiDPI的方法有很多，下面列出我尝试过的几种：
- 使用一键开启脚本
一键开启脚本是使用起来最方便的方式，但我在用的时候总是找不到想要的Hi-DPI分辨率。
https://github.com/syscl/Enable-HiDPI-OSX
https://github.com/xzhih/one-key-hidpi
- 手动修改配置文件
手动修改配置文件也有好几种方法：
  1. [黑果小兵](https://blog.daliansky.net/)的教程  
  [使用HIDPI解决睡眠唤醒黑屏、花屏及连接外部显示器的正确姿势](https://blog.daliansky.net/Use-HIDPI-to-solve-sleep-wake-up-black-screen,-Huaping-and-connect-the-external-monitor-the-correct-posture.html)  
  这种方法需要用到FixEDID这个软件来生成edid的配置文件，然后进行替换。FixEDID这个软件一开始我没法打开，用Xcode重新生成后就可以正常使用了。
  2. 在线配置文件编辑器  
  [SCALED RESOLUTIONS](https://comsysto.github.io/Display-Override-PropertyList-File-Parser-and-Generator-with-HiDPI-Support-For-Scaled-Resolutions/)
  这种方法是把填入的配置信息转换成plist文件中的专用格式，完成后导出，在系统中替换。我用的时候添加了1600x900分辨率，但是最后并没有1600x900的HiDPI分辨率。
  3. <span id='jump'>使用hackintool生成配置文件，进行替换。</span>
  打开hackintool，选择音频菜单，hackintool会识别到显示器，点击下方的添加分辨率按钮，就会自动生成不同的分辨率。点击导出按钮，就会生成配置文件和内核扩展。我采用的是这种方法，但只替换了配置文件，没有用生成的内核扩展。这种方法也无法生成1600x900的HiDPI分辨率，但它的效果是最好的。我用其他方法时，有的会使屏幕颜色变得很偏红，有的连1440x810的HiDPI分辨率都没有。

# 一个大坑
HiDPI技术适合用在高分屏上，Dell P2417H这块1080P的屏幕开HiDPI也能改善视觉效果，但是只能开低于1920x1080的HiDPI，这就导致了屏幕的内容会比原来放大。理想的分辨率是1600x900，但是我的两块屏幕就只有一块能开这个分辨率，另一块在这个分辨率下不是HiDPI，这可能是显卡的问题。  
上也有人反映了[跟我一样的问题](https://github.com/xzhih/one-key-hidpi/issues/37)，给出的解决方案是将CPU仿冒成skylake微架构的。这种方法我试了之后没有作用，而且还开不了机，电脑启动后无法进入登录界面，整个屏幕都黑了，只有右上角有一个光标在闪。费了好大劲才把这个问题解决了，但开机之后有一块屏幕的颜色严重偏红。用了[上面手动修改方法三](#jump)才把屏幕颜色给调正常了。

# 一点感想  
装黑苹果的门槛是越来越低了，但是一旦出现问题有时还是手足无措，只能照着网上的方案一个个去试，运气好可能就成功了。期间要花费大量的时间和精力，但好像没什么收获，感觉没学到多少东西。要真正从解决问题中成长，应该尝试去了解问题根源，不能只停留在表象。这次的问题跟EDID是逃不了干系了，但我对EDID这东西不了解，只能盲目地一个方法一个方法去试，这样的效率必然是极低的。
