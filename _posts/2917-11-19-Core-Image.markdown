---
layout: post
title:  "使用 Core Image 制作海报"
date:   2017-11-19 17:00:00 +0800
categories: iOS CoreImage
location: Beijing, China
tags: 笔记
---

之前做了个以图片的形式分享菜谱作品等到微信的需求，使用 Core Image 完成，效果图如下：

<img src="http://ww1.sinaimg.cn/mw690/62dbaf47ly1flnit4vuflj20u01tddpp.jpg" width="480">

Core Image 除了用来做人脸识别之外，最主要的用途就是做图片处理了，Core Image 提供了 CIImage 和 CIFilter 来描述这个过程。整个过程很直观，CIImage 是代表 image 的 data model，CIFilter 即图片处理方法（滤镜）。苹果在 Core Image 框架里面内置了二百多种 Filter，足以满足大多数需求，如果需要自定义 Filter，Core Image 插件化的架构也可以很方便的把我们自己实现的 Filter 集成进来。如何自定义 Filter 可以看这里[Creating Custom Filters](https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/CoreImaging/ci_custom_filters/ci_custom_filters.html)。

对图片做 Transform、Crop、rotate 等操作也是 Filter 里面的一类，CIImage 提供了一些列 imageBy 方法对这些常用的操作进行了封装，方便调用。在做分享海报的过程中，大量用到了这些操作。

### 设置画布

需要先确定最后生成的图片的宽度，以像素为单位，这里先定为 1080 px，排版需要用。然后先生成一张白色的画布 renderImage：

```objc
CGFloat renderWidth = 1080.0f;
CIImage *renderImage = [CIImage imageWithColor:[[CIColor alloc] initWithColor:[UIColor whiteColor]]];
```
此时 renderImage 的尺寸是无限大，以它 {0,0} 的位置作为海报左下角布局。

### 绘制切了圆角的正方形头像

```objc
UIImage *avatarImage = /* 事先准备好的头像图片 */;
CIImage *avatar = [CIImage imageWithCGImage:avatarImage.CGImage]; // 生成 CIImage

// 画圆
CGFloat radius = avatar.extent.size.width / 2;
NSDictionary *maskParas = @{@"inputCenter"  : [CIVector vectorWithX:radius Y:radius],
                            @"inputRadius0" : @(radius),
                            @"inputRadius1" : @(radius),
                            @"inputColor0" : [CIColor colorWithRed:1 green:1 blue:1 alpha:1],
                            @"inputColor1" : [CIColor colorWithRed:0 green:0 blue:0 alpha:1]};
CIImage *circle = [CIFilter filterWithName:@"CIRadialGradient"
                       withInputParameters:maskParas].outputImage;

// 生成圆形 mask
CIImage *mask = [CIFilter filterWithName:@"CIMaskToAlpha"
                     withInputParameters:@{kCIInputImageKey : circle}].outputImage;

// 生成新的切了圆角的头像
avatar = [CIFilter filterWithName:@"CIBlendWithAlphaMask"
              withInputParameters:@{kCIInputMaskImageKey : mask,
                                        kCIInputImageKey : avatar}].outputImage;

// 移动头像到画布上设计好的位置，假设左下角坐标是 {x,y}
avatar = [avatar imageByApplyingTransform:CGAffineTransformMakeTranslation(x, y)];

// 最后画到画布上，得到新的画布
renderImage = [avatar imageByCompositingOverImage:renderImage];

```

```- imageByCompositingOverImage:``` 实际上是使用了名为 CISourceOverCompositing 的 Filter，将两张图片合成一张。
后面的其他部分都是这样，先生成一个单独的 CIImage，再移动到画布上相应的位置，最后使用 ```- imageByCompositingOverImage:``` 合成新的 renderImage。从下往上画，只要从最后添加的 CIImage 的 extent 用 CGRectGetMaxY 就可以知道现在画了多高了。（这里的坐标系和 UIKit 的相反，Y坐标的方向是由下向上的）

### 绘制文本

绘制文本需要先把文本转成 image，剩下的和其他部分没有区别。转换代码：
```c
/*
 * text  text to be drawn
 * width max width in px
 * scale iphone screen scale factor
 */
static UIImage* h_createImageFromAttributedString(NSAttributedString *text,CGFloat width,CGFloat scale) {
    CGRect drawRect = [text boundingRectWithSize:CGSizeMake(width / scale, CGFLOAT_MAX)
                                         options:NSStringDrawingUsesLineFragmentOrigin|NSStringDrawingUsesFontLeading
                                         context:nil];
    drawRect.origin = CGPointZero;
    UIGraphicsBeginImageContextWithOptions(drawRect.size, NO, scale);
    [text drawInRect:drawRect];
    UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    return image;
}
```
其中 scale 使用 ```[UIScreen mainScreen].scale``` 即可。

### 绘制二维码

Core Image 内置了用于生成二维码的 Filter CIQRCodeGenerator。

```objc
NSString *link = @"https://www.google.com";
NSData *codeData = [link dataUsingEncoding:NSUTF8StringEncoding];
CIFilter *codeFilter = [CIFilter filterWithName:@"CIQRCodeGenerator"
                            withInputParameters:@{@"inputMessage" : codeData,
                                                @"inputCorrectionLevel" : @"Q"}];
CIImage *image = codeFilter.outputImage;
```

参数中的 inputCorrectionLevel 指生成的二维码的纠错级别，LMQH 四个级别可选，级别越高纠错能力越强。注意最后生成的 CIImage 还需要使用 imageByApplyingTransform 缩放到计算好的尺寸。

### 生成海报

一步步把海报各个部分挨个“画”上去，最后计算出 renderHeight，使用 CIContext 生成海报。
```objc
CGRect renderRect = CGRectMake(0, 0, canvasWidth, renderHeight);
// 切成最后计算好的尺寸    
renderImage = [renderImage imageByCroppingToRect:renderRect];
    
// draw image
CIContext *context = [CIContext contextWithOptions:@{kCIContextUseSoftwareRenderer : @NO}];
CGFloat scale = [UIScreen mainScreen].scale;
CGImageRef imageRef = [context createCGImage:renderImage fromRect:renderImage.extent];
UIImage *finalImage = [UIImage imageWithCGImage:imageRef scale:scale orientation:UIImageOrientationUp];
CGImageRelease(imageRef);
```
得到最终的海报图片。

### 关于 CIContext
1. kCIContextUseSoftwareRenderer 为 NO 的时候会使用 GPU 来处理，只能在主线程运行。
2. CIContext 的创建开销比较大，尽量重用。
3. 在用 CIFilter 处理 CIImage 的时候，并没有真正对图片数据进行处理，只是描述了整个过程；等到 CIContext 来生成的图片的时候，才真正处理了图片。并且 Core Image 会对一些过程做合并优化，提高效率。

参考资料:[Core Image Getting the Best Performance](https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/CoreImaging/ci_performance/ci_performance.html#//apple_ref/doc/uid/TP30001185-CH10-SW1)