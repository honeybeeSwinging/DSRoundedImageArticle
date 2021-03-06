# DSImageViewRound <a href="#使用方法">使用方法</a>
**iOS图片高性能设置圆角**

一般我们在iOS开发的过程中设置圆角都是如下这样设置的。

     avatarImageView.clipsToBounds = YES;
     [avatarImageView.layer setCornerRadius:50];
     
     这样设置会触发离屏渲染，比较消耗性能。比如当一个页面上有十几头像这样设置了圆角
     会明显感觉到卡顿。
     
     注意：png图片UIImageView处理圆角是不会产生离屏渲染的。（ios9.0之后不会离屏渲染，ios9.0之前还是会离屏渲染）。

------

所以如果要高性能的设置圆角就需要找另外的方法了。下面是我找到的一些方法

![IMG_1816.PNG](http://upload-images.jianshu.io/upload_images/101810-9c34cd972e319727.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**设置圆角的方法**

- 直接使用setCornerRadius

    > 这种就是最常用的，也是最耗性能的。

- setCornerRadius设置圆角之后，shouldRasterize=YES光栅化

    > avatarImageView.clipsToBounds = YES;
      [avatarImageView.layer setCornerRadius:50];
      avatarImageView.layer.shouldRasterize = YES;
      
    > shouldRasterize=YES设置光栅化，可以使离屏渲染的结果缓存到内存中存为位图，
      使用的时候直接使用缓存，节省了一直离屏渲染损耗的性能。

    > 但是如果layer及sublayers常常改变的话，它就会一直不停的渲染及删除缓存重新
      创建缓存，所以这种情况下建议不要使用光栅化，这样也是比较损耗性能的。


- 直接覆盖一张中间为圆形透明的图片

    > 这种方法就是多加了一张透明的图片，GPU计算多层的混合渲染blending也是会消耗
     一点性能的，但比第一种方法还是好上很多的。

- UIImage drawInRect绘制圆角

    > 这种方式GPU损耗低内存占用大，而且UIButton上不知道怎么绘制，可以用
       UIimageView添加个点击手势当做UIButton使用。

    > 
      ```objectivec
      UIGraphicsBeginImageContextWithOptions(avatarImageView.bounds.size, NO, [UIScreen mainScreen].scale);
      [[UIBezierPath bezierPathWithRoundedRect:avatarImageView.bounds cornerRadius:50] addClip];
      [image drawInRect:avatarImageView.bounds];
      avatarImageView.image = UIGraphicsGetImageFromCurrentImageContext();
      UIGraphicsEndImageContext();
      ```
      
    > 这段方法可以写在SDWebImage的completed回调里，在主线程异步绘制。
      也可以封装到UIImageView里，写了个DSRoundImageView。后台线程异步绘制，不会阻塞主线程。
      
- SDWebImage处理图片时Core Graphics绘制圆角
 
    > //UIImage绘制为圆角
      ```objectivec
      int w = imageSize.width;
      int h = imageSize.height;
      int radius = imageSize.width/2;  
      ```
      ```objectivec
      UIImage *img = image;
      CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
      CGContextRef context = CGBitmapContextCreate(NULL, w, h, 8, 4 * w, colorSpace, kCGImageAlphaPremultipliedFirst);
      CGRect rect = CGRectMake(0, 0, w, h);
      ```
      ```objectivec
      CGContextBeginPath(context);
      addRoundedRectToPath(context, rect, radius, radius);
      CGContextClosePath(context);
      CGContextClip(context);
      CGContextDrawImage(context, CGRectMake(0, 0, w, h), img.CGImage);
      CGImageRef imageMasked = CGBitmapContextCreateImage(context);
      img = [UIImage imageWithCGImage:imageMasked];
      ```
      ```objectivec
      CGContextRelease(context);
      CGColorSpaceRelease(colorSpace);
      CGImageRelease(imageMasked);
      ```
     以上代码我写成了UIImage的类别:UIImage+DSRoundImage.h
     并在SDWebImage库里处理image的时候使用类别方法绘制圆角并缓存。

-------

**使用Instruments的Core Animation查看性能**
- Color Offscreen-Rendered Yellow

    > 开启后会把那些需要离屏渲染的图层高亮成黄色，这就意味着黄色图层可能存在性能问题。

- Color Hits Green and Misses Red

    > 如果shouldRasterize被设置成YES，对应的渲染结果会被缓存，如果图层是绿色，就表示这些缓存被复用；如果是红色就表示缓存会被重复创建，这就表示该处存在性能问题了。

用Instruments测试得
- 第一种方法，ios9.0之前UIImageView和UIButton都高亮为黄色。ios9.0之后只有UIButton高亮为黄色。

- 第二种方法UIImageView和UIButton都高亮为绿色

- 第三种方法，无任何高亮，说明没离屏渲染。
  这种圆片覆盖的方法一般只用在底色为纯色的时候，如果圆角图片的父View是张图片的时候就没办法了，而且底色如果是多种颜色的话那   要做多张不同颜色的圆片覆盖。（可以用代码取底色的颜色值给圆片着色）

- 第四种方法无任何高亮，说明没离屏渲染（但是CPU消耗和内存占用会很大）
 
- 第五种方法无任何高亮，说明没离屏渲染，而且内存占用也不大。(暂时感觉是最优方法)

# 使用方法
[查看具体使用demo](https://github.com/walkdianzi/DSRoundedImageDemo)

# 最后
- 如果我的项目对你有帮助欢迎 Star
- 如果在使用过程中遇到BUG，希望你能Issues我
- 如果在使用过程中发现功能不够用或者想交流的，希望你能Issues我，或者联系我QQ：398411773
- 如果你想为DSRoundedImage输出代码，请拼命Pull Requests我
