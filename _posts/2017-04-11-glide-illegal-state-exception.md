---
layout: post
title: Glide加载图片出现java.lang.IllegalStateException
tags: [Glide, IllegalStateException, recycled-bitmap]
---

## Glide简介
[Glide][1]是面向Android的快速高效的开源媒体管理和图像加载框架，提供媒体解码，内存和磁盘缓存以及资源池等简单易用的用户体验。

## Glide用法
### gradle依赖
```gradle
repositories {
  mavenCentral() // jcenter() works as well because it pulls from Maven Central
}

dependencies {
  compile 'com.github.bumptech.glide:glide:3.7.0'
  compile 'com.android.support:support-v4:19.1.0'
}
```

### 代码片段
```java
// For a simple view:
@Override public void onCreate(Bundle savedInstanceState) {
  ...
  ImageView imageView = (ImageView) findViewById(R.id.my_image_view);

  Glide.with(this).load("http://goo.gl/gEgYUd").into(imageView);
}

// For a simple image list:
@Override public View getView(int position, View recycled, ViewGroup container) {
  final ImageView myImageView;
  if (recycled == null) {
    myImageView = (ImageView) inflater.inflate(R.layout.my_image_view, container, false);
  } else {
    myImageView = (ImageView) recycled;
  }

  String url = myUrls.get(position);

  Glide
    .with(myFragment)
    .load(url)
    .centerCrop()
    .placeholder(R.drawable.loading_spinner)
    .crossFade()
    .into(myImageView);

  return myImageView;
}
```

### ProGuard混淆规则
{% raw %}
```xml
-keep public class * implements com.bumptech.glide.module.GlideModule
-keep public enum com.bumptech.glide.load.resource.bitmap.ImageHeaderParser$** {
  **[] $VALUES;
  public *;
}

# for DexGuard only
-keepresourcexmlelements manifest/application/meta-data@value=GlideModule
```
{% endraw %}

### 兼容性
Glide详细的兼容性问题可查看Github仓库，这里只列举一个：
- **圆角图片**
[CircleImageView][3]/[CircularImageView][4]/[RoundedImageView][5] 与TransitionDrawable (.crossFade() 配合 .thumbnail() 或 .placeholder()使用时) 和 animated GIFs配合使用时会出现已知的[问题][2]，解决方法是使用[BitmapTransformation][6] (.circleCrop() 在v4版本才支持) 或者 .dontAnimate()。

## Can't call reconfigure() on a recycled bitmap
### 问题原因
>This error usually means that the bitmap pool is tainted, so if you're using Transformations anywhere in the app, that could be a culprit. Someone is recycling Bitmaps that are in the pool, outside of Glide's knowledge.

Transformations anywhere in the app could be a culprit.
App中任何地方出现的Transformations可能就是罪魁祸首，Transformation就是配合Glide实现圆角图片的API。
### 典型代码
```java
public class BitmapFixedWidthTransform extends BitmapTransformation {

    public BitmapFixedWidthTransform(Context context) {
        super(context);
    }

    @Override
    protected Bitmap transform(BitmapPool pool, Bitmap toTransform, int outWidth, int outHeight) {
        int targetWidth = outWidth;

        double aspectRatio = (double) toTransform.getHeight() / (double) toTransform.getWidth();
        int targetHeight = (int) (targetWidth * aspectRatio);
        Bitmap result = Bitmap.createScaledBitmap(toTransform, targetWidth, targetHeight, true);
        if (result != toTransform) {
            // Same bitmap is returned if sizes are the same
            toTransform.recycle(); // removed this
        }
        return result;
    }

    @Override
    public String getId() {
        return "transformation" + " desiredWidth";
    }
}
```

### 解决方法
解决方法其实已经在上面的代码注释中，既然是由recycled bitmap引起的，那么去掉下面的代码就OK：
```java
toTransform.recycle(); // removed this
```

---------

[1]: https://github.com/bumptech/glide
[2]: https://github.com/bumptech/glide/issues?q=is%3Aissue+CircleImageView+OR+CircularImageView+OR+RoundedImageView
[3]: https://github.com/hdodenhof/CircleImageView
[4]: https://github.com/Pkmmte/CircularImageView
[5]: https://github.com/vinc3m1/RoundedImageView
[6]: https://github.com/wasabeef/glide-transformations
