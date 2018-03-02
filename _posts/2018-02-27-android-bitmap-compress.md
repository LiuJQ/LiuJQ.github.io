---
layout: post
title: Android -- Bitmap加载性能优化
tags: [Android, Bitmap, 性能优化]
---

## 前言
&emsp;&emsp;图片是移动端开发中司空见惯的内容，开发者多数时候都会涉及图片加载的功能。Bitmap是图片在Android中的一种承载方式，它可以被加载到需要展示图片的地方，比如ImageView或是View的background。随着移动设备的更新换代，图片的分辨率越来越高，加载Bitmap的时候就不得不考虑OOM的问题了。

## BitmapFactory
&emsp;&emsp;Android中Bitmap的加载一般是通过BitmapFactory类来实现的。BitmapFactory类提供多种加载方式，常用的几个加载方式：

**文件**
```java
public static Bitmap decodeFile(String pathName, Options opts) {
    ...
}
```
**资源**
```java
public static Bitmap decodeResource(Resources res, int id, Options opts) {
    ...
}
```
**字节数组**
```java
public static Bitmap decodeByteArray(byte[] data, int offset, int length, Options opts) {
    ...
}
```
**数据流**
```java
public static Bitmap decodeStream(InputStream is, Rect outPadding, Options opts) {
    ...
}
```

## Problem
&emsp;&emsp;通常，直接调用BitmapFactory的上述几个方法，传递相对应的参数就可以加载Bitmap了。Bitmap对象在Android应用程序中占用的内存是非常高:

> 一张480x800的图片，Bitmap占用内存计算：
>
| 类型 | 内存计算 | 内存占用(KB) |
| :----: | :----: | :----: |
|ARGB_8888 |	480×800×4÷1024	|	1500
|ARGB_4444 |	480×800×2÷1024	|	750
|ARGB_565 |	480×800×2÷1024	|	750
|ARGB_8 |	480×800×1÷1024	|	375

&emsp;&emsp;不作任何处理，直接调用BitmapFactory的decode方法来加载Bitmap会有几个问题：
* 图片分辨率高，Bitmap加载耗时长
* 图片分辨率高，Bitmap占用内存很高，可能出现OOM
* 图片数量很多，Bitmap占用内存很高，可能出现OOM

## Solution
&emsp;&emsp;不知道大家注意到没有，上述几个decode方法都有一个共同的参数——Options，它其实是BitmapFactory的一个内部类，主要提供Bitmap加载配置参数。我们今天要讨论的优化点就是从Options配置参数出发，减少Bitmap加载过程中的内存消耗，降低Bitmap加载耗时，尽量降低OOM出现的风险。

**Options参数源码**

&emsp;&emsp;提取Options关键配置参数，看看源码如何描述的：
```java
/**
 * If set to true, the decoder will return null (no bitmap), but
 * the out... fields will still be set, allowing the caller to query
 * the bitmap without having to allocate the memory for its pixels.
 */
public boolean inJustDecodeBounds;

/**
 * If set to a value > 1, requests the decoder to subsample the original
 * image, returning a smaller image to save memory. The sample size is
 * the number of pixels in either dimension that correspond to a single
 * pixel in the decoded bitmap. For example, inSampleSize == 4 returns
 * an image that is 1/4 the width/height of the original, and 1/16 the
 * number of pixels. Any value <= 1 is treated the same as 1. Note: the
 * decoder uses a final value based on powers of 2, any other value will
 * be rounded down to the nearest power of 2.
 */
public int inSampleSize;
```

**Options参数分析**
```java
/**
 * 如果为true，不会执行decode操作，返回bitmap为null，因此不会分配内存，但是会返回图片的分辨率参数
 */
public boolean inJustDecodeBounds;

/**
 * 图片采样率，影响图片长宽和像素密度
 */
public int inSampleSize;
```
&emsp;&emsp;结合两个参数的含义，要降低Bitmap内存占用，需要根据期望图片尺寸和原图尺寸来计算最佳图片采样率，使得图片在不失真的情况下内存占用尽量小，然后使用计算出的最佳图片采样率来decode图片bitmap，达到降低Bitmap内存占用及减少decode操作耗时的目的。

&emsp;&emsp;那么，如何能够快速又不消耗内存地得到原图尺寸呢？答案就是利用inJustDecodeBounds参数！设置Options的inJustDecodeBounds参数为true，此时执行一次decode操作即可在不消耗内存的条件小拿到原图尺寸。

### 解决方案

&emsp;&emsp;结合上述Options参数分析，我们可以写出一个简单的解决方案：
```xml
1. 创建BitmapFactory.Options对象options
2. 设置options参数inJustDecodeBounds为true
3. 执行一次decode操作，获取到原图尺寸options.outWidth和options.outHeight
4. 按照原图长宽比，计算得出期望图片尺寸
5. 根据原图尺寸和期望尺寸，计算出最佳图片采样率
6. 设置options参数inJustDecodeBounds为false，设置options参数为最佳图片采样率
7. 再次执行decode操作，得到优化内存占用后Bitmap对象
```

#### 计算期望图片尺寸
&emsp;&emsp;如何计算呢？

&emsp;&emsp;假设我们最终要decode的bitmap尺寸是长(maxWidth)和宽(maxHeight)，然后根据原图尺寸的长宽比来按比例缩放计算，得到缩放后的期望尺寸desiredWidth和desiredHeight。为啥要这么做呢？因为预设的maxWidth和maxHeight，它们的比例并不一定跟原图比例相符，按比例缩放后的期望尺寸才能保证最终图片不变形。

&emsp;&emsp;具体算法如下：
```java
private static int getResizedDimension(int maxPrimary, int maxSecondary, int actualPrimary, int actualSecondary) {
    // If no dominant value at all, just return the actual.
    if (maxPrimary == 0 && maxSecondary == 0) {
        return actualPrimary;
    }

    // If primary is unspecified, scale primary to match secondary's scaling ratio.
    if (maxPrimary == 0) {
        double ratio = (double) maxSecondary / (double) actualSecondary;
        return (int) (actualPrimary * ratio);
    }

    if (maxSecondary == 0) {
        return maxPrimary;
    }

    double ratio = (double) actualSecondary / (double) actualPrimary;
    int resized = maxPrimary;
    if (resized * ratio > maxSecondary) {
        resized = (int) (maxSecondary / ratio);
    }
    return resized;
}
```

#### 计算最佳图片采样率
&emsp;&emsp;根据前面对inSampleSize参数的解析，它的最终取值为2的指数倍。结合期望尺寸和原图尺寸的比值，我们可以计算出一个最接近该比值的采样率数值，把它作为最佳采样率。
> Powers of 2 : 2^n
<br/>2^0 = 1
<br/>2^1 = 2
<br/>2^2 = 4
<br/>2^3 = 8
<br/>2^4 = 16
<br/>2^5 = 32
<br/>2^6 = 64
<br/>2^7 = 128
<br/>2^8 = 256
<br/>2^9 = 512
<br/>2^10 = 1024

&emsp;&emsp;具体算法如下：
```java
/**
 * 计算最佳图片采样率
 *
 * @param actualWidth   原图长度
 * @param actualHeight  原图宽度
 * @param desiredWidth  目标长度
 * @param desiredHeight 目标宽度
 * @return  best sample ratio.
 */
private static int findBestSampleSize(int actualWidth, int actualHeight, int desiredWidth, int desiredHeight) {
    double wr = (double) actualWidth / desiredWidth;
    double hr = (double) actualHeight / desiredHeight;
    double ratio = Math.min(wr, hr);
    float n = 1.0f;
    while ((n * 2) <= ratio) {
        n *= 2;
    }
    return (int) n;
}
```

### 最终实现
&emsp;&emsp;上述的解决方案经过我们的分解之后，一步一步实现了，最终的代码实现如下：
```java
public static boolean compress(Context context, BitmapCompressOptions compressOptions) {
    String filePath = !TextUtils.isEmpty(compressOptions.sourceFilePath) ? compressOptions.sourceFilePath : getFilePath(context, compressOptions.sourceUri);
    if (TextUtils.isEmpty(filePath)) {
        return false;
    }

    BitmapFactory.Options options = new BitmapFactory.Options();

    options.inJustDecodeBounds = true;
    BitmapFactory.decodeFile(filePath, options);
    int actualWidth = options.outWidth;
    int actualHeight = options.outHeight;
    int desiredWidth = getResizedDimension(compressOptions.maxWidth, compressOptions.maxHeight, actualWidth, actualHeight);
    int desiredHeight = getResizedDimension(compressOptions.maxHeight, compressOptions.maxWidth, actualHeight, actualWidth);

    options.inJustDecodeBounds = false;
    options.inSampleSize = findBestSampleSize(actualWidth, actualHeight, desiredWidth, desiredHeight);

    Bitmap bitmap;
    Bitmap destBitmap = BitmapFactory.decodeFile(filePath, options);

    // If necessary, scale down to the maximal acceptable size.
    if (destBitmap.getWidth() > desiredWidth || destBitmap.getHeight() > desiredHeight) {
        bitmap = Bitmap.createScaledBitmap(destBitmap, desiredWidth, desiredHeight, true);
        destBitmap.recycle();
    } else {
        bitmap = destBitmap;
    }

    return null != compressOptions.destFile && compressBitmap(compressOptions, bitmap);
}
```

## 结语
&emsp;&emsp;随着Android系统越来越高度成熟，各种开源项目也层出不穷，在图片加载这块上就已经有很多优秀的开源项目了，比如早期的[Android-Universal-Image-Loader](https://github.com/nostra13/Android-Universal-Image-Loader)，FackBook开源的[Fresco](https://github.com/facebook/fresco)以及Google开发人员开源的[Glide](https://github.com/bumptech/glide)（现已被Google采用）。

&emsp;&emsp;今天分享的Bitmap加载性能优化方案，也可以运用到项目工程中，比如图片上传功能模块，既可以减少选择图片后的等待耗时，又可以上传压缩后的图片文件，达到性能和体验双收目的。
