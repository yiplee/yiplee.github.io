---
layout: post
title:  "使用 Core Image 制作海报"
date:   2017-11-19 17:00:00 +0800
categories: iOS Cocoa
location: Beijing, China
tags: 笔记
---

半年前做了个以图片的形式分享菜谱作品等到微信的需求，效果图如下：
![作品海报](http://ww1.sinaimg.cn/mw690/62dbaf47ly1flnit4vuflj20u01tddpp.jpg)

接到这个需求之后我就打算用 Core Image 来做。Core Image 除了用来做人脸识别之外，最主要的用途就是做图片处理了，Core Image 提供了 CIImage 和 CIFilter 来描述这个过程。整个过程很直观，CIImage 是代表 image 的 data model，CIFilter 即“滤镜”,